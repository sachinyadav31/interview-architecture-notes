# 07: Product search with Elasticsearch

## What it does
Marketplace and catalog search (full-text, filters by category/brand, price stats, discounts, sustainability fields). MySQL is the **source of truth**; Elasticsearch is a **read-optimised copy**.

Two services:
- **`ep-indexing-service`**: writes to ES (event-driven + bulk refresh APIs).
- **`ep-search-api-service`**: reads from ES (query building, pagination, aggregations).

## Indexing pipeline
```mermaid
flowchart LR
  PS[product-service<br/>save product] -->|ep.acme.product-index-topic<br/>{entityType, entityIds[]}| K[(Kafka)]
  K --> L[ProductIndexEventListener<br/>TenantAwareKafkaListener]
  L --> S[IndexJobScheduler.submitIndexingJob]
  S --> F[IndexJobProcessorFactory<br/>pick processor by entityType]
  F --> V[validateBefore<br/>index name, schema path, stored proc configured?]
  V --> C[createIndex if missing<br/>name = acme_products<br/>mapping from JSON schema file]
  C --> P[partition IDs into batches<br/>Guava Lists.partition batchSize]
  P --> SP[MySQL stored procedure<br/>CALL get_products_json(csvIds)<br/>returns one JSON doc per row]
  SP --> B[Bulk index into ES]
```

### Key classes
- `ProductIndexEventListener` / `CatalogProductsIndexEventListener` / `ProductCategoryIndexEventListener` / `ProductBrandIndexEventListener`: one per entity type, all `TenantAwareKafkaListener`.
- `IndexJobScheduler.submitIndexingJob(event)`: orchestrates: processor → validate → create index if needed → batch → process.
- `IndexJobProcessorFactory`: **Factory pattern** mapping `IndexEventEntityType` (PRODUCT, CATALOG_PRODUCT, CATEGORY, BRAND) → processor.
- `DefaultIndexJobProcessor` (abstract, Template Method): `validateBefore`, `getDataSourceInformation`, `loadData`, `getData`. The **index name = `<tenantQualifier>_<indexName>`**, so each tenant's data sits in its own index.
- `EsConfigProperties.indicesConfigurations`: per entity: `indexName`, `indexSchemaPath` (a JSON mapping in resources), `storedProcName`, `storedProcInputProp`.
- `ESDataIndexManager`: uses the **new Elasticsearch Java API client** (`ElasticsearchClient` over `RestClientTransport` + `JacksonJsonpMapper`): `indices().exists/create/delete`, bulk index. `createIndex(name, schemaPath, dropIfExist)` loads the mapping with `.withJson(InputStream)`.
- **Bulk refresh API**: `ProductBulkDataRefreshController` (`AbstractBulkDataRefreshController`) re-indexes everything (`getAllEntityIds` → batches), and can e-mail a report when done (`RefreshEmailProcessorFactory`). Used after mapping changes.

### Why a stored procedure builds the JSON
The product document joins many tables (product, prices, category path, brand, attributes, supplier, discounts, sustainability). One stored proc returns **ready-made JSON per product**, so the Java side stays generic (the same processor for every entity type; only config changes). Trade-off: logic hidden in SQL, harder to test and version. I updated the product stored proc for **sustainability fields** in 2026, and added **discount percentage** indexing in 2024 for the marketplace.

## Search side
- `ProductSearchRequestHandler`, `ProductCategorySearchRequestHandler`, `ProductBrandSearchRequestHandler` build `SearchRequest`s with `BoolQueryBuilder`:
  - `multiMatchQuery` for free text across name/description/SKU;
  - `termQuery`/`termsQuery` for exact filters (category ID, brand, supplier, status);
  - `nestedQuery` for nested attributes/prices;
  - `AggregationBuilders.stats` (min/max/avg price for the price slider), plus `nested` aggregations for facets.
- Pagination: from/size (1-indexed at the API like the rest of the platform).
- Tenant: the index name is built from the tenant context, so a tenant can only search its own index.

## Weak spots and how I'd improve them
- **Errors swallowed** in listeners: a failed index call is just logged, so search silently goes stale. Fix: retry + DLT + a reconciliation job (compare DB `updated_at` vs ES).
- **Drop-and-recreate** reindex causes downtime. Fix: **index aliases**: build `acme_products_v2`, then atomically switch alias `acme_products` from v1 to v2 (zero downtime).
- Deep pagination with from/size is expensive above 10k. Use `search_after` with a sort tiebreaker.
- One index per tenant → many small shards. For many small tenants, a shared index with a `tenant` field + filtered aliases (or routing by tenant) uses resources better.

## Interview Q&A

**Q: Why Elasticsearch and not just MySQL `LIKE`?**
A: Full-text relevance (analyzers, stemming, fuzzy matching), fast facets/aggregations, and offloading read traffic from the transactional DB. `LIKE '%x%'` can't use indexes and has no relevance ranking.

**Q: How do you keep ES in sync with MySQL?**
A: Event-driven: when a product changes, the product service publishes the IDs on Kafka; the indexing service reloads those rows (stored proc) and bulk-indexes them. Plus a bulk refresh API for full rebuilds. It's eventually consistent: a second or two of lag. Alternative: CDC with Debezium reading the binlog, which doesn't depend on every code path remembering to publish.

**Q: How did you make search multi-tenant?**
A: Index per tenant: `<qualifier>_products`. The indexing and search services build the name from the ThreadLocal tenant context. Superset dashboards also point at `<qualifier>_*` indices (see [11](11-tenant-onboarding.md)).

**Q: What's a mapping, and why define it yourself?**
A: The schema of an index: field types and analyzers. Dynamic mapping guesses wrong (e.g. IDs as text, prices as long). We keep explicit JSON mappings in resources: `keyword` for exact filters and aggregations, `text` with an analyzer for search, `nested` for arrays of objects that must be queried together (attribute name + value).

**Q: match vs term query?**
A: `match` analyses the input (lowercase, tokenise) and is used on `text` fields for full-text search. `term` is an exact value on `keyword`/numeric fields with no analysis. Using `term` on a `text` field is a classic bug.

**Q: What's a nested type, and why use it?**
A: Arrays of objects are flattened by default, so `{color:red,size:L},{color:blue,size:M}` would match "red AND M". `nested` keeps each object separate so queries match within one object. It costs more (hidden documents).

**Q: How would you reindex without downtime?**
A: Create a new versioned index with the new mapping, bulk-load it (from DB or `_reindex`), then atomically swap an alias the app reads from. Delete the old index later.

**Q: Bulk API: why?**
A: One HTTP request carrying many index operations, far fewer round trips. We batch IDs (configurable `batchSize`) so each bulk call is a reasonable size (a few MB). Check the per-item errors in the bulk response; a bulk call can partly fail.
