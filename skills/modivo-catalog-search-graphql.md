---
name: modivo-catalog-search-graphql
description: >-
  Search and browse the MODIVO catalog over the storefront GraphQL API — faceted product search with
  aggregations, category tree traversal, and the omnibus (EU lowest-30-day) price fields — with no
  credentials.
api: MODIVO Storefront GraphQL API
base_url: https://modivo.pl/graphql
generated: '2026-08-12'
method: generated
source: graphql/modivo-storefront.graphql
operations:
  - Query.products
  - Query.categoryList
  - Query.category
  - Query.categories
  - Query.cmsPage
  - Query.getSearchLinkFromDeepLink
---

# Catalog search on MODIVO (GraphQL)

**This is the capability MODIVO's REST contract does not have.** There is no product search on REST.
If you need to find a product, you are on GraphQL, and you need no credentials to do it.

## Endpoint

`POST https://modivo.pl/graphql` with `Content-Type: application/json`. Introspection is open — you
can pull the live schema yourself with a standard introspection query, and a copy taken on
2026-08-12 is in `graphql/modivo-storefront.graphql`.

Send a real `User-Agent`. Both MODIVO hosts are behind Cloudflare and a default library UA drew an
HTTP 403 HTML challenge from the sibling eobuwie endpoint during profiling.

An optional `Store` request header selects the country/language store view — the endpoint declares
`Vary: Store`, so caching by URL alone is wrong.

## Steps

1. **Search products.**

   `products(search: String, filter: ProductAttributeFilterInput, pageSize: Int, currentPage: Int,
   sort: ProductAttributeSortInput): Products`

   The `Products` result carries `items`, `total_count`, `page_info`, `aggregations`, `filters`,
   `sort_fields` and `suggestions`. Ask for `aggregations` on the first call: it tells you which
   facets are actually available for this result set, so you can build the second, narrower query
   from real data instead of guessing filter names.

2. **Page through.** `pageSize` and `currentPage`, with `page_info { page_size current_page
   total_pages }` in the selection set. This is offset pagination, not cursors — a page walk over a
   changing catalog can skip or repeat items. For a full harvest, sort by a stable field and record
   what you have seen.

3. **Browse the tree instead.** `categoryList(filters: CategoryFilterInput, pageSize: Int,
   currentPage: Int): [CategoryTree]` when you want structure rather than a query. `category` and
   `categories` are the single-node variants.

4. **Read prices carefully.** The schema carries `omnibus_price`, `formatted_omnibus_price` and
   `omnibus_discount_amount` alongside the normal price fields. These implement the EU Omnibus
   Directive's lowest-price-in-30-days disclosure. If you are surfacing a discount to a user in the
   EU, the omnibus fields are the ones that matter legally — a percentage computed against the
   regular price alone is the wrong number.

5. **Resolve an app or campaign link.** `getSearchLinkFromDeepLink` turns a MODIVO deep link into a
   search, and `appDeepLink` goes the other way. Useful when you are handed a MODIVO URL and need the
   query behind it.

6. **Hand the SKU to checkout.** Take `items[].sku` from the search result into
   `modivo-guest-checkout` — the REST guest-cart flow needs a SKU and has no way to find one.

## Reading the response

- **A GraphQL error does not always change the HTTP status.** Once a query parses and validates,
  resolver failures come back as **HTTP 200** with a populated `errors[]` array and `null` for the
  failed field. Always inspect `errors[]`, never just the status code.
- Branch on `errors[].extensions.code` (`NO_SUCH_ENTITY`, `FORBIDDEN`, …). It is stable and
  unlocalized. The `message` is not.
- Only authorization failures escalate the status — an unauthenticated `customer` query returns
  HTTP 403 with `extensions.category: graphql-authorization`.

## Runtime signals

Every response carries `x-request-id`, `x-correlation-id` and `x-causation-id`. Log all three.
They are the only support handle this provider gives you, and the REST surface does not return them
at all.

There are no rate-limit headers. Nothing tells you your remaining budget. Keep concurrency low, and
treat any non-JSON body as a Cloudflare challenge rather than an API response.
