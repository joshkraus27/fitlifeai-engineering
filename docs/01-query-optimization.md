# Optimizing search over a 745K-row food database: three rounds

FitLifeAI's food logging is built on the USDA food database — about **745,000 rows** of foods, branded products, and nutrient data stored in Supabase (PostgreSQL). Users search it constantly: every meal log starts with a text search, including voice-based logging where the transcribed phrase becomes the query.

This is the story of making that search fast — twice — because performance is a property you have to keep measuring, not a fix you apply once.

## Round 1: ~9s → ~130ms with a GIN trigram index

Search originally took **~9 seconds** per query on warm cache. The root cause: pattern-matching (`ILIKE '%chicken%'`) against the `name` column with nothing to accelerate it. A B-tree index can't help queries that match *inside* a string — B-trees are sorted by whole values, so there's no way to seek into them by an interior substring. PostgreSQL sequentially scanned 745K rows on every keystroke.

A **GIN (Generalized Inverted Index)** works like the index at the back of a textbook: it maps tokens to the rows containing them. With the `pg_trgm` extension, the tokens are **trigrams** — every 3-character sequence in a string — which makes substring and fuzzy matching indexable:

```sql
CREATE INDEX food_library_name_trgm_idx
  ON public.food_library USING gin (name gin_trgm_ops);
```

Warm query latency dropped from **~9s to ~130ms**. (Re-measured recently: the core indexed query runs at **123ms warm**, all buffer hits, zero disk reads.)

## Round 2: search became a system

The latency fix exposed a *relevance* problem: branded products ("TYSON CHICKEN BREAST") were outranking plain USDA entries for generic queries — exactly wrong for voice logging, where the user says a generic phrase and expects the generic food.

The search layer grew into a set of purpose-built indexes:

```sql
-- Full-text search over a tsvector column
CREATE INDEX food_library_name_fts_idx
  ON public.food_library USING gin (name_fts);

-- Fast prefix matching for type-ahead
CREATE INDEX food_library_name_lower_pattern_idx
  ON public.food_library USING btree (lower(name) text_pattern_ops);

-- Partial trigram indexes segmenting canonical USDA foods vs. branded products
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

And the query became a **two-tier ranked search** running as a single PostgreSQL function (called via Supabase RPC):

- **Tier 0 — canonical USDA foods** (SR Legacy, Foundation, FNDDS): matched by full-text query (`name_fts @@ websearch_to_tsquery(...)`) *or* trigram similarity (`name % q`), ranked by exact-prefix match, then `ts_rank_cd`, then trigram `similarity()`, then name length (shorter = more generic).
- **Tier 2 — everything else**, including branded foods.

Tiers are unioned and ordered tier-first, so generic foods always beat branded ones — unless the caller passes `prefer_branded => true`, which inverts the tier order.

## Round 3: the regression I found by re-measuring

While preparing this write-up, I re-ran `EXPLAIN ANALYZE` on the production function and found it had quietly regressed to **~1.5s warm**. The core query was still 123ms — but the two-tier function built around it had grown slow, and nobody had re-profiled it.

The plan pointed at one line:

```
Bitmap Index Scan on food_library_name_trgm_idx
  Index Cond: (name % 'chicken breast'::text)
  ... rows=79000 ... actual time=496.807
```

The trigram similarity operator (`%`) was matching **79,000 rows** for a two-word query — roughly a tenth of the database — because `pg_trgm`'s default similarity threshold (0.3) is very loose for multi-word strings. The function then scanned that bloated candidate set once per tier and spent the rest of its time on index rechecks.

The fix was one line inside the function:

```sql
PERFORM set_limit(0.5);  -- tighten pg_trgm similarity threshold for this call
```

(Supabase's managed Postgres doesn't permit `ALTER FUNCTION ... SET pg_trgm.similarity_threshold`, so the function sets it at call time via `set_limit()`.)

### Measured results

| Configuration | Trigram candidates | Warm execution time |
|---|---|---|
| Threshold 0.3 (old default) | 79,000 rows | 866ms – 1.57s |
| Threshold 0.5 (shipped) | 41,364 rows | **232ms** |

```
-- threshold 0.5, warm:
Planning Time: 6.675 ms
Execution Time: 231.671 ms
```

**Recall check before shipping:** I compared typo-query results ("chiken brest") at thresholds 0.3, 0.4, and 0.5 — the returned rows were **identical** at all three. The tightened threshold cost zero recall on the fuzzy-match path; it only excluded candidates that were never going to rank anyway.

## What the profiling also surfaced

Reading the actual result rows during the recall check exposed a **data-quality issue**: obvious branded products (e.g., "TYSON CHICKEN BREAST") have `data_type = NULL` instead of `'branded_food'` — meaning they bypass the branded tier logic and the partial branded index entirely. The query architecture is right; some of the data isn't labeled to match it. Backfilling `data_type` for these rows is queued as cleanup.

Also identified for future work: typo queries currently surface only short-named (mostly branded) matches, because trigram similarity penalizes the length gap between a two-word typo and long canonical USDA names ("Chicken, broilers or fryers, breast, meat only, roasted"). This predates the threshold change (verified — same behavior at 0.3). The structural fix is matching against a normalized short-name column rather than raw USDA names.

## What I learned

- **Measure before guessing, and keep measuring after.** The original 9s and the 1.5s regression were both found the same way: timing the real query. The system evolved past its last measurement — performance is a property, not a patch.
- **Index type matters more than index existence.** B-trees for equality/prefix, GIN trigram for substring/fuzzy, GIN over tsvector for full-text. A real search feature needs several, matched to access patterns.
- **`EXPLAIN ANALYZE` tells you exactly where the time goes.** One line of plan output (79K rows from a similarity scan) located the regression; one line of SQL fixed it.
- **Verify the trade before shipping it.** Tightening the threshold *could* have hurt fuzzy recall; comparing result rows across thresholds proved it didn't. Fast and wrong is still wrong — check both axes.
- **Reading your own results finds bugs your metrics won't.** The mislabeled `data_type` rows only surfaced because I looked at what the query actually returned, not just how fast it returned it.
