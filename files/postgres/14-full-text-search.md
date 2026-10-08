# Full-Text Search

> TL;DR: PG's FTS is built around `tsvector` (parsed/normalized documents) and `tsquery` (parsed queries). Use `to_tsvector(config, text)` to build documents and the `@@` operator to match. GIN index a generated `tsvector` column. Good enough for most apps; reach for Elasticsearch when you need cross-language analyzers, distributed search, or BM25-grade ranking at scale.

## Core types & operators

```sql
SELECT to_tsvector('english', 'The quick brown foxes jumped over lazy dogs.');
-- 'brown':3 'dog':9 'foxes':4 'jumped':5 'lazy':8 'quick':2
-- Notice stemming (foxes -> fox? actually 'foxes' kept; lemma depends on dictionary) and stopword removal.

SELECT to_tsquery('english', 'quick & brown');
SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('english', 'quick & fox'); -- true
```

Query operators inside `tsquery`:

```
&    AND
|    OR
!    NOT
<->  followed-by (phrase)
<N>  followed-by N words later
```

Phrase search needs `plainto_tsquery`/`phraseto_tsquery` (PG 9.6+) or `websearch_to_tsquery` (PG 11+):

```sql
SELECT websearch_to_tsquery('english', '"machine learning" -tutorial');
-- 'machin' <-> 'learn' & !'tutori'
```

`websearch_to_tsquery` accepts Google-ish syntax (`"phrase"`, `-exclude`, `OR`) and is safer for user input.

## A real schema

```sql
CREATE TABLE articles (
  id     bigserial PRIMARY KEY,
  title  text NOT NULL,
  body   text NOT NULL,
  search tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')),  'B')
  ) STORED
);

CREATE INDEX idx_articles_search ON articles USING gin (search);
```

`setweight` lets you boost certain fields (A > B > C > D) at ranking time.

## Querying & ranking

```sql
SELECT id, title,
       ts_rank(search, q) AS rank,
       ts_headline('english', body, q, 'MaxFragments=2,MinWords=5,MaxWords=15') AS snippet
FROM articles, websearch_to_tsquery('english', :q) AS q
WHERE search @@ q
ORDER BY rank DESC
LIMIT 20;
```

- `ts_rank` — frequency-based score, with weight class support.
- `ts_rank_cd` — cover-density (Clarke et al.) considering distance between matches.

## Configurations

`'english'`, `'simple'`, `'spanish'`, etc. The config drives:
- Parser (how text is tokenized).
- Dictionaries (stopwords, stemming).

```sql
SELECT cfgname FROM pg_ts_config;
```

`simple` does no stemming/stopwords — useful for usernames, codes.

## Phrase, prefix, similarity

```sql
-- prefix
SELECT * FROM articles WHERE search @@ to_tsquery('postgr:*');

-- phrase
SELECT * FROM articles WHERE search @@ phraseto_tsquery('quick brown fox');

-- "similar to" with pg_trgm (NOT a tsvector thing)
CREATE EXTENSION pg_trgm;
SELECT title, similarity(title, 'postgres tutorial') AS s
FROM articles
WHERE title % 'postgres tutorial'
ORDER BY s DESC;
```

`pg_trgm` is great for misspelling-tolerant search on short fields (names, titles). FTS is better for longer documents.

## Highlighting

```sql
SELECT ts_headline('english', body, websearch_to_tsquery('english', :q),
                   'StartSel=<b>, StopSel=</b>, MinWords=5, MaxWords=20')
FROM articles WHERE search @@ websearch_to_tsquery('english', :q);
```

## Indexing strategies

- **GIN** — slower to build, faster to search. Default for most FTS.
- **GiST** — faster to update, slower to search; useful when index is small or rapidly changing.

Add `fastupdate = off` on GIN for read-heavy workloads to avoid pending-list scans on read.

## Multi-language

Store the language alongside the document, choose config per row:

```sql
CREATE TABLE docs (
  id bigserial,
  lang regconfig NOT NULL,
  body text,
  search tsvector GENERATED ALWAYS AS (to_tsvector(lang, body)) STORED
);
```

Querying:

```sql
SELECT * FROM docs WHERE search @@ to_tsquery(lang, :q);
```

## Custom dictionaries

Synonyms, stopword files, ispell dictionaries — manage with `CREATE TEXT SEARCH DICTIONARY`/`CONFIGURATION`. Beyond interview scope; mention you know they exist.

## PG FTS vs Elasticsearch

| | PG FTS | Elasticsearch |
|---|--------|--------------|
| Transactional consistency with relational data | Yes | No (eventual via outbox) |
| Cross-language analyzers | Limited | Excellent |
| Distributed shards | Need partitioning / Citus | Built-in |
| Relevance tuning (BM25, custom scoring) | Basic ts_rank | Strong |
| Ops complexity | Same DB | Extra cluster |
| Throughput at >100M docs | Workable, tuning required | Designed for this |

**Rule of thumb**: PG FTS is enough until search becomes a core product feature with relevance / typo / multi-language demands.

## Performance tips

- Store the `tsvector` as a generated column (PG 12+) — no triggers needed.
- GIN index that column.
- Use `websearch_to_tsquery` for user input (won't blow up on weird syntax).
- Limit before highlighting (`ts_headline` is expensive).
- For very large corpora, partition by time and use partial indexes.

## Interview Questions

**Q1. Difference between `tsvector` and `tsquery`?**
`tsvector` is a parsed, normalized representation of a document (lexeme + positions + weight). `tsquery` is a parsed query (lexemes plus operators).

**Q2. How would you build a FTS pipeline on PG?**
Add a `tsvector` generated column with `setweight` per field, GIN-index it, query with `@@` and `websearch_to_tsquery`, rank with `ts_rank`, highlight with `ts_headline`.

**Q3. GIN vs GiST for FTS?**
GIN: faster searches, slower writes, larger. GiST: smaller, faster updates, slower searches. GIN is the typical default.

**Q4. How do you do typo-tolerant search?**
PG FTS alone doesn't fuzzy-match. Use `pg_trgm` (trigram similarity) on short fields, or combine FTS with trigram, or move to Elasticsearch.

**Q5. When does PG FTS stop being good enough?**
When you need rich linguistic analyzers (CJK, Arabic), BM25-style relevance, distributed scaling beyond what partitioning supports, or full custom scoring — switch to Elasticsearch / Vespa.

**Q6. Why does FTS sometimes return rows in unexpected order?**
Default is by `rank`, but if you ORDER BY ID or omit ORDER BY entirely, the index returns physical order. Always ORDER BY `ts_rank` (or `ts_rank_cd`) for relevance.

## Common Pitfalls

- Forgetting to specify a language config -> uses `default_text_search_config` (sometimes `simple`), losing stemming.
- Forgetting to GIN-index the `tsvector` column — sequential scans on millions of rows.
- Letting users send raw tsquery syntax (`!&|`) — use `websearch_to_tsquery` or `plainto_tsquery`.
- Highlighting before LIMIT — expensive.
- Storing language as `text` and casting on every query.

## See also

- [Indexes (GIN/GiST)](06-indexes.md)
- [JSONB & arrays](13-jsonb-and-arrays.md)
- [System design: search](../system-design/high-level-design/03-databases.md)
