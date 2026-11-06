# ADR-0003: PostgreSQL FTS over Elasticsearch

- **Status**: Accepted
- **Date**: 2024 (decision), recorded 2026-05-08
- **Deciders**: Michael DAŞ

## Context

Hallego is a classifieds marketplace where ad discovery via free-text
search is a primary user flow. Ads have a title, a description, and
structured filters (category, price, location). The search must:

1. Handle Turkish-specific characters and stemming (ı, ğ, ş, ç, ö, ü)
2. Combine free-text matching with geospatial filtering (PostGIS)
   and structured filters (price range, category, attributes)
3. Be reasonably fast at the expected scale (tens of thousands of
   active ads, growing)
4. Stay within a single-VPS infrastructure budget

The team is one developer. Operating an additional stateful service
in production has a non-trivial maintenance cost.

## Decision

We use **PostgreSQL Full-Text Search** with the built-in `turkish`
text search configuration as the search backend. Search uses `tsvector` columns
generated from title and description with weight differentiation
(title weighted higher than description).

The search query combines:
- `tsvector @@ tsquery` for free-text relevance
- PostGIS `ST_DWithin` for radius-based geographic filtering
- Standard SQL predicates for category, price, attribute filters

Search history and saved searches are stored in the same database.

## Alternatives Considered

### Option A: Elasticsearch / OpenSearch
The industry-standard search engine. Best-in-class relevance,
aggregations, faceting, and Turkish analyzers.

**Rejected because:**
- Adds a stateful service to operate, monitor, back up, and upgrade
  alongside PostgreSQL and Redis. Triples the database operational
  surface.
- Indexes must be kept in sync with the source of truth (PostgreSQL).
  This requires either a CDC pipeline or manual reindex on every
  write, both add complexity and failure modes.
- Memory footprint is significant on a single Hetzner VPS shared with
  the app, queue worker, and Mercure.
- Geographic filtering combined with full-text is awkward across two
  systems (search returns IDs, then PostGIS filters them, or vice
  versa, neither is efficient).

### Option B: Meilisearch / Typesense
Lightweight, modern search engines with simpler operations than
Elasticsearch.

**Rejected because:**
- Same fundamental issue: a separate service to keep in sync with
  PostgreSQL.
- Geographic + full-text combination still requires coordinating two
  systems.
- Turkish language support is less mature than PostgreSQL's built-in `turkish` configuration.

### Option C: Trigram search (`pg_trgm`)
Pure PostgreSQL approximate matching using trigram similarity.

**Rejected because:**
- No native Turkish stemming, it matches surface-level character
  similarity, not linguistic roots.
- Acceptable as a complement (e.g. for username search or typo
  tolerance) but not as the primary search engine.

## Consequences

### Positive
- **Single source of truth**: search queries the same database as
  every other read path. No sync, no eventual consistency.
- **Trivial JOINs across search and structured filters**: a search
  with category, price range, geographic radius, and free-text terms
  is a single SQL query with one execution plan.
- **PostgreSQL handles Turkish well enough**: the `turkish` text search
  configuration provides stemming, the `unaccent` extension is installed, and
  the database image sets the `tr_TR.UTF-8` locale (with an ICU `turkish_ci`
  collation for sorting and comparison).
- **One less service to operate**: no Elasticsearch cluster to
  upgrade, back up, or monitor.
- **Search history and saved searches use normal foreign keys**:
  no cross-system references.

### Negative
- **Relevance tuning is more limited than Elasticsearch**: BM25,
  custom analyzers, learning-to-rank, and synonym graphs are not
  available out of the box. We use `ts_rank` with weighted
  tsvectors, which is good enough for current scale but ceiling-bound.
- **Faceted search aggregations are less efficient**: counting ads
  per category for a given query requires either separate queries
  or window functions, less elegant than Elasticsearch aggs.
- **Search performance is bounded by PostgreSQL**: at very high
  scale (100k+ ads, hundreds of QPS), we will need to revisit. The
  likely migration path is to extract a search service that
  consumes a logical replication stream.

### Neutral
- The Turkish locale is set up in the PostGIS-based Docker image
  (`tr_TR.UTF-8`). Onboarding a new ops engineer requires reading the
  Dockerfile.

## References

Paths below refer to the private repository.

- [PostgreSQL Text Search documentation](https://www.postgresql.org/docs/17/textsearch.html)
- `backend/database/migrations/`, search
  index migrations
- `backend/app/search/`, search module
- ARCHITECTURE.md § Technical Decisions
