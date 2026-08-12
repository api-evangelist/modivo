---
name: modivo-guest-checkout
description: >-
  Run a complete anonymous guest checkout against the MODIVO storefront — create a cart, add a line,
  price it, choose shipping and payment, and place the order — using MODIVO's own live REST contract.
api: MODIVO Commerce REST API
base_url: https://modivo.pl/rest/all
generated: '2026-08-12'
method: generated
source: openapi/modivo-commerce-rest-api-openapi.yml
operations:
  - PostV1Guestcarts
  - PostV1GuestcartsCartIdItems
  - GetV1GuestcartsCartId
  - PostV1GuestcartsCartIdBillingaddress
  - PostV1GuestcartsCartIdEstimateshippingmethods
  - PostV1GuestcartsCartIdShippinginformation
  - GetV1GuestcartsCartIdPaymentmethods
  - PutV1GuestcartsCartIdSelectedpaymentmethod
  - GetV1GuestcartsCartIdTotals
  - PutV1GuestcartsCartIdOrder
---

# Guest checkout on MODIVO

The guest-cart family is the only part of MODIVO's REST contract that works with **no credentials at
all**. Everything here is anonymous.

## Before you start

- Base: `https://modivo.pl/rest/{store}` — `{store}` is the Adobe Commerce store view. `all` and
  `default` both resolve.
- No `Authorization` header is needed for any step below.
- **There is no idempotency on this API.** The final step creates a real order and cannot be safely
  retried. Read `conventions/modivo-conventions.yml` before you automate it.
- Errors come back as the Magento envelope with **Polish** message text. Branch on the HTTP status
  and on `parameters[].fieldName`, never on the message string. See
  `errors/modivo-problem-types.yml`.

## Steps

1. **Create the cart.** `POST /V1/guest-carts` (`PostV1Guestcarts`). The response body is the masked
   cart id — an opaque string. Every subsequent call takes it as `{cartId}`. Persist it: it is the
   only handle you have, and it is also your only dedupe key at step 9.

2. **Add a line.** `POST /V1/guest-carts/{cartId}/items` (`PostV1GuestcartsCartIdItems`) with a
   `cartItem` carrying `sku`, `qty` and `quote_id` (the masked id). You need a real SKU — the REST
   contract has no product search. Get one from `Query.products` on
   `https://modivo.pl/graphql`, or from `GET /V1/products-render-info` if you already have product
   ids. Do **not** use `GET /V1/search`; despite the name it is Magento's generic search engine, it
   is ACL-protected, and it returns 401 anonymously.

3. **Read the cart back.** `GET /V1/guest-carts/{cartId}` (`GetV1GuestcartsCartId`) returns the full
   quote. Read `extension_attributes` as well as the base fields — MODIVO puts duty calculation and
   in-store sales data there, and a client that ignores it drops real data.

4. **Set the billing address.** `POST /V1/guest-carts/{cartId}/billing-address`
   (`PostV1GuestcartsCartIdBillingaddress`).

5. **Estimate shipping.** `POST /V1/guest-carts/{cartId}/estimate-shipping-methods`
   (`PostV1GuestcartsCartIdEstimateshippingmethods`) with a candidate address. Returns available
   carrier/method pairs with prices. Each entry has `available` and `error_message` — check both;
   an entry can be present and still not selectable.

6. **Optionally apply a coupon.** `PUT /V1/guest-carts/{cartId}/coupons/{couponCode}`
   (`PutV1GuestcartsCartIdCouponsCouponCode`). A bad code returns 400, not 404.

7. **Commit shipping information.** `POST /V1/guest-carts/{cartId}/shipping-information`
   (`PostV1GuestcartsCartIdShippinginformation`) with the chosen `carrier_code` and `method_code`.
   This is the most efficient call in the whole flow: it returns the payment methods **and** the
   totals in one response, where the GraphQL surface needs two mutations plus a query. Use its
   response instead of doing steps 8 and 9 separately if you can.

8. **Confirm payment method.** `GET /V1/guest-carts/{cartId}/payment-methods`
   (`GetV1GuestcartsCartIdPaymentmethods`), then `PUT
   /V1/guest-carts/{cartId}/selected-payment-method`
   (`PutV1GuestcartsCartIdSelectedpaymentmethod`).

9. **Verify totals, then place the order.** `GET /V1/guest-carts/{cartId}/totals`
   (`GetV1GuestcartsCartIdTotals`) and check `grand_total` against what you expect. Then `PUT
   /V1/guest-carts/{cartId}/order` (`PutV1GuestcartsCartIdOrder`), which returns the order id.

## Rules for step 9

- **Never auto-retry.** No `Idempotency-Key` is accepted. A retried call can create a duplicate
  order.
- On a timeout or a 5xx with no body, treat the outcome as **unknown**. Resolve it by reading order
  state (`Query.guestOrderByToken` on GraphQL), not by calling again.
- Dedupe on the masked cart id. A cart converts to exactly one order; if you already hold an order id
  for that cart id, you are done.
- Expect no rate-limit header to tell you when to back off. Both hosts are behind Cloudflare — if you
  get an HTML body instead of JSON, you have been challenged, not rate-limited in-band. Stop and back
  off.
