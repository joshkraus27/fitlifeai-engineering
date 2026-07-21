# Optimizing search over a 745K-row food database: ~9s → ~130ms

FitLifeAI's food logging is built on the USDA food database — about **745,000 rows** of foods, branded products, and nutrient data stored in Supabase (PostgreSQL). Users search it constantly: every meal log starts with a text search, including voice-based logging where the transcribed phrase becomes the query.

## The problem

Search was taking **~9 seconds** per query on warm cache. For a feature users hit dozens of times a day, that's not slow — it's broken.

The root cause: the search was pattern-matching against the `name` column with nothing to accelerate it. A B-tree index can't help queries that match *inside* a string (`ILIKE '%chicken%'`) — B-trees are sorted by whole values, so there's no way to seek into them by an interior substring. PostgreSQL had no choice but to sequentially scan 745K rows on every search.

## The fix, round one: a GIN trigram index

A **GIN (Generalized Inverted Index)** works like the index at the back of a textbook: instead of sorting whole values, it maps tokens to the rows containing them. With the `pg_trgm` extension, the tokens are **trigrams** — every 3-character sequence in the string — which makes substring and fuzzy matching indexable:

```sql
CREATE INDEX food_library_name_trgm_idx
  ON public.food_library USING gin (name gin_trgm_ops);
```

A search for "chicken" becomes a handful of trigram lookups and an intersection instead of 745K row comparisons. Warm query latency dropped from **~9s to ~130ms**.

## The fix, round two: search became a system

The single index fixed latency but exposed a *relevance* problem: branded products ("CHICKEN BREAST NUGGETS, Brand X") were outranking the plain USDA entry for "chicken breast" — exactly wrong for voice logging, where the user says a generic phrase and expects the generic food.

The search layer grew into a set of purpose-built indexes:

```sql
-- Full-text search over a tsvector column
CREATE INDEX food_library_name_fts_idx
  ON public.food_library USING gin (name_fts);

-- Fast prefix matching for type-ahead
CREATE INDEX food_library_name_lower_pattern_idx
  ON public.food_library USING btree (lower(name) text_pattern_ops);

-- Partial trigram indexes segmenting the search space:
-- canonical USDA foods vs. branded products
CREATE INDEX food_library_name_trgm_canonical
  ON public.food_library USING gin (lower(name) gin_trgm_ops)
  WHERE usda_fdc_id IS NOT NULL
    AND (data_type IS NULL OR data_type <> 'branded_food');

CREATE INDEX food_library_name_trgm_branded
  ON public.food_library USING gin (lower(name) gin_trgm_ops)
  WHERE data_type = 'branded_food';

-- Popularity ordering
CREATE INDEX food_library_log_count_idx
  ON public.food_library USING btree (log_count DESC);
```

The **partial indexes** are the interesting move: rather than one index over everything, canonical and branded foods get separate, smaller indexes matching how the query actually treats them.

## The query: two-tier ranked search

Search runs as a single PostgreSQL function (called via Supabase RPC). It queries in two tiers:

- **Tier 0 — canonical USDA foods** (SR Legacy, Foundation, FNDDS): matched by full-text query (`name_fts @@ websearch_to_tsquery(...)`) *or* trigram similarity (`name % q`), ranked by exact-prefix match, then `ts_rank_cd`, then trigram `similarity()`, then name length (shorter = more generic = better).
- **Tier 2 — everything else**, including branded foods.

Tiers are unioned and ordered tier-first, so generic foods always beat branded ones — unless the caller passes `prefer_branded => true`, which inverts the tier order for the flows where a branded match is what the user wants.

```sql
CREATE OR REPLACE FUNCTION public.search_food_library(q text, prefer_branded boolean DEFAULT false)
 RETURNS SETOF food_library
 LANGUAGE sql STABLE
AS $function$
  WITH tok AS (
    SELECT websearch_to_tsquery('english', q) AS tsq
  ),
  gen AS (
    SELECT f AS row, ts_rank_cd(f.name_fts, tok.tsq) AS rk, 0 AS tier
    FROM food_library f, tok
    WHERE (f.name_fts @@ tok.tsq OR f.name % q)
      AND f.data_type IN ('sr_legacy_food','foundation_food','survey_fndds_food')
    ORDER BY (lower(f.name) LIKE lower(q) || '%')::int DESC,
             rk DESC, similarity(f.name, q) DESC, length(f.name) ASC
    LIMIT 60
  ),
  oth AS (
    SELECT s.row, ts_rank_cd((s.row).name_fts, tok.tsq) AS rk, 2 AS tier
    FROM (
      SELECT f AS row
      FROM food_library f, tok
      WHERE (f.name_fts @@ tok.tsq OR f.name ILIKE '%' || q || '%' OR f.name % q)
        AND (f.data_type IS NULL OR (prefer_branded AND f.data_type = 'branded_food'))
      LIMIT 300
    ) s, tok
  )
  SELECT (u.row).*
  FROM (
    SELECT row, rk, tier FROM gen
    UNION ALL
    SELECT row, rk, tier FROM oth
  ) u
  ORDER BY
    (CASE WHEN prefer_branded THEN 2 - u.tier ELSE u.tier END) ASC,
    (lower((u.row).name) LIKE lower(q) || '%')::int DESC,
    u.rk DESC,
    length((u.row).name) ASC
  LIMIT 60
$function$
```

## Results

| Metric | Before | After |
|---|---|---|
| Warm query latency | ~9,000 ms | ~130 ms |
| Speedup | — | ~70x |
| Relevance | branded foods outranked generics | generic-first, branded opt-in |

<!-- PASTE ZONE: run the two EXPLAIN ANALYZE commands Claude gave you and paste
     the output in the fenced block below, then delete this comment. -->
```
[EXPLAIN ANALYZE output goes here]
```

## What I learned

- **Measure before guessing.** The 9s number came from timing the real query. The first fix was cheap once the cause was proven.
- **Index type matters more than index existence.** B-trees for equality/prefix, GIN trigram for substring/fuzzy, GIN over tsvector for full-text — the index has to match the access pattern, and a real search feature ends up needing several.
- **Fast and wrong is still wrong.** The latency fix surfaced the ranking problem. Search quality is a product decision (generic beats branded for voice logging) that has to be encoded in the query — and the schema (partial indexes) can be shaped to serve that decision.
- **GIN indexes trade write speed for read speed.** Every insert updates many token entries. For a read-heavy reference dataset like USDA foods, that trade is free.
