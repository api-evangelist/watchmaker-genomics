---
name: watchmaker-genomics-browse-catalog
description: >-
  Search and read the Watchmaker Genomics reagent catalog — NGS library prep kits, enzymes,
  polymerases, reverse transcriptases and MDx tools — through the storefront GraphQL endpoint.
  Read-only. No credential required.
api: Watchmaker Genomics Storefront GraphQL API
endpoint: https://www.watchmakergenomics.com/graphql
auth: none
operations:
  - products
  - categories
  - route
  - search
  - storeConfig
  - cmsPage
generated: '2026-09-04'
method: generated
source: graphql/watchmaker-genomics-commerce.graphql
---

# Browse the Watchmaker Genomics catalog

Watchmaker Genomics publishes no developer documentation. Its storefront nonetheless serves a
fully introspectable Adobe Commerce GraphQL endpoint over its own product catalog, anonymously.
Every field named below was read from the schema that endpoint returns, and the queries were run
against the live host.

## Endpoint

```
POST https://www.watchmakergenomics.com/graphql
Content-Type: application/json
```

No API key, no bearer token, no signup. The endpoint sets `PHPSESSID` and
`private_content_version` cookies on every POST — discard them; nothing here needs a session.

## Search the catalog

`products(search: String, filter: ProductAttributeFilterInput, pageSize: Int, currentPage: Int, sort: ProductAttributeSortInput): Products`

```graphql
{
  products(search: "polymerase", pageSize: 20, currentPage: 1) {
    total_count
    page_info { current_page page_size total_pages }
    items {
      sku
      name
      url_key
      stock_status
      price_range { minimum_price { final_price { value currency } } }
    }
  }
}
```

SKUs are Watchmaker catalog numbers and are sometimes a hyphen-joined family covering several
pack sizes — `7K0116-7K0117-7K0118-7K0120-7K0121` is StellarTaq DNA Polymerase. Do not split them.

## Fetch one product

Use `filter` rather than `search` when you already know the identifier:

```graphql
{
  products(filter: { sku: { eq: "7K0113" } }) {
    items {
      sku
      name
      description { html }
      short_description { html }
      media_gallery { url label }
      price_tiers { quantity final_price { value currency } }
      categories { uid name url_path }
    }
  }
}
```

## Walk the category tree

`categories(filters: CategoryFilterInput, pageSize: Int, currentPage: Int): CategoryResult`

```graphql
{
  categories(filters: { url_key: { eq: "enzymes" } }) {
    items { uid name url_path products(pageSize: 50) { total_count items { sku name } } }
  }
}
```

`category` and `categoryList` are both `@deprecated` in this schema in favour of `categories`.
Do not use them.

## Resolve a storefront URL

`route(url: String!): RoutableInterface` turns any page path into the entity behind it:

```graphql
{ route(url: "products/enzymes/stellarscripthtplus.html") { relative_url type __typename } }
```

`urlResolver` is `@deprecated`; use `route`.

## Read store configuration

`storeConfig` answers anonymously and is worth reading before any write:

```graphql
{ storeConfig { store_name base_url default_display_currency_code locale is_guest_checkout_enabled order_cancellation_enabled returns_enabled } }
```

On 2026-09-04 this returned `order_cancellation_enabled: false` and `returns_enabled: "disabled"`.

## Pagination

`pageSize` and `currentPage` on the input; `total_count` and `page_info { current_page page_size
total_pages }` on the output. There is no cursor.

## Errors

GraphQL's own `errors[]` envelope, not the REST one. Branch on
`extensions.category`: `graphql-input`, `graphql-no-such-entity`, `graphql-authorization`,
`graphql-authentication`. A malformed document returns HTTP 500 with a syntax error in `errors[]`
rather than a 400.

## What is NOT here

There is no REST equivalent of any of this. The Swagger document at
`/rest/all/schema?services=all` contains no product and no category operation — only a
price-render helper and a generic quick search. If you need the catalog, GraphQL is the only way in.

## Rate limits

None are published and none were observed. A full introspection of the 636-type schema was served
without throttling. Assume an undeclared edge limit exists and back off on any 429 or 503, which
will arrive with no `Retry-After`.
