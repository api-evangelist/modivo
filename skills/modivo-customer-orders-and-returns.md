---
name: modivo-customer-orders-and-returns
description: >-
  Authenticate as a MODIVO customer and read order history, track a return, and raise a new return —
  all of which exist only on the GraphQL surface and have no REST equivalent.
api: MODIVO Storefront GraphQL API
base_url: https://modivo.pl/graphql
generated: '2026-08-12'
method: generated
source: graphql/modivo-storefront.graphql
operations:
  - Mutation.generateCustomerToken
  - Query.customer
  - Query.customerOrders
  - Query.customerOrdersExt
  - Query.customerReturnOrders
  - Query.customerReturnReasons
  - Query.customerReturnCarriers
  - Mutation.addReturn
  - Mutation.cancelOrder
  - Query.guestOrderByToken
---

# Customer orders and returns on MODIVO

Order history and returns are **GraphQL-only**. MODIVO's published REST contract contains no
customer-order operation at all — it can create an order but cannot read one back. Anyone profiling
MODIVO from the Swagger document alone will wrongly conclude this capability does not exist.

## 1. Get a token

```
mutation { generateCustomerToken(email: "...", password: "...") { token } }
```

Signature: `generateCustomerToken(email: String!, password: String!): CustomerToken`.

Social variants exist for Apple and other providers (`generateCustomerTokenWithApple` and
siblings) if you are brokering an existing session rather than a password.

Send it as `Authorization: Bearer <token>` on every subsequent request. There is no OAuth, no
refresh-token flow and no scope model — this is a bare credential exchange. Treat the token as a
full-account bearer credential and store it accordingly.

## 2. Confirm the session

```
query { customer { firstname lastname email } }
```

An unauthenticated or expired token returns **HTTP 403** with
`extensions.category: "graphql-authorization"`, `extensions.code: "FORBIDDEN"` and
`data.customer: null`. This is the one case where MODIVO's GraphQL escalates the HTTP status; most
other failures return 200.

## 3. Read order history

`customerOrders: CustomerOrders` takes no arguments and returns `items`, `total_count`, `page_info`
and `date_of_first_order`. `customerOrdersExt` and `customerOrderExt` are MODIVO's extended variants
— check both; the extended shapes carry provider-specific fields the base type does not.

For a guest order you hold a token for, use `guestOrderByToken` instead — no customer login needed.
This is also how you resolve an uncertain outcome after a timeout on order placement, since REST
order placement is not idempotent and must never be retried.

## 4. Return an item

1. `customerReturnOrders(input: FindCustomerReturnInput!): CustomerReturnOrder` — find the order and
   the items eligible for return.
2. `customerReturnReasons` — fetch the allowed reason codes. Do not hard-code them; they are store-
   view specific.
3. `customerReturnCarriers` — fetch the carriers the customer may use.
4. `addReturn(input: AddCustomerReturnInput!): Boolean` — raise the return. Note the return type is a
   bare `Boolean`, so a successful call tells you nothing about the return that was created. Re-read
   `customerReturnOrders` afterwards to get the return number.

MODIVO's returns provider pushes the return number back into the order asynchronously, through the
inbound webhook receiver `POST /V1/my-return-webhook/add-return-number-to-order` (see
`asyncapi/modivo-webhooks.yml`). That means the return number is **not** available immediately after
`addReturn` succeeds. Poll `customerReturnOrders` rather than assuming it is there.

## 5. Cancel an order

`cancelOrder`, with `confirmCancelOrder` as the confirmation step. Both are destructive and neither
is idempotent.

## Things that will bite you

- **No push, no polling budget.** MODIVO publishes no outbound events, no webhook subscription, and
  no rate limits or rate-limit headers. If you need to know when an order or return changes state,
  polling is the only option, and nothing tells you how hard you may poll. Be conservative.
- **`Boolean` return types hide failure detail.** `addReturn` and `addProductReview` both return a
  bare boolean. Always re-read the entity.
- **Deprecations are in the schema, not in a policy.** 385 fields in this schema carry `@deprecated`
  with a reason. MODIVO publishes no deprecation policy, no changelog and no sunset headers, so
  diffing the SDL in `graphql/modivo-storefront.graphql` against a fresh introspection is the only
  advance warning you will get.
- **`deleteCustomer` exists and is irreversible.** Do not expose it to an agent.
