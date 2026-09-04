---
name: watchmaker-genomics-quote-a-basket
description: >-
  Price a basket of Watchmaker Genomics reagents — build a guest cart, add SKUs, and estimate
  shipping and totals for a delivery address — WITHOUT placing an order. Stops deliberately short
  of placeOrder, because an order placed on this store cannot be cancelled through the API.
api: Watchmaker Genomics Storefront GraphQL API
endpoint: https://www.watchmakergenomics.com/graphql
auth: none
operations:
  - createGuestCart
  - addProductsToCart
  - setGuestEmailOnCart
  - estimateShippingMethods
  - estimateTotals
  - removeItemFromCart
  - clearCart
generated: '2026-09-04'
method: generated
source: >-
  graphql/watchmaker-genomics-commerce.graphql,
  conventions/watchmaker-genomics-conventions.yml
---

# Quote a basket of Watchmaker reagents

This skill covers the safe part of the checkout surface: everything up to, and not including, the
irreversible step. Field names and argument shapes are read from the schema the endpoint serves.

## Read this before you write anything

Queried live on 2026-09-04:

```graphql
{ storeConfig { order_cancellation_enabled returns_enabled } }
# -> { "order_cancellation_enabled": false, "returns_enabled": "disabled" }
```

`cancelOrder` and `requestReturn` exist in the schema. They are switched off on this store. **An
order placed here cannot be taken back programmatically.** Recovery is a human path:
orders@watchmakergenomics.com or +1-720-543-2174. There is also no idempotency key anywhere on
this surface — a retried `placeOrder` is a second order.

So: quote, then hand the quote to a person. Do not call `placeOrder` from an agent without
explicit human approval per order.

## 1. Open a guest cart

`createGuestCart(input: CreateGuestCartInput): CreateGuestCartOutput`

```graphql
mutation { createGuestCart { cart { id } } }
```

Keep `cart.id` — it is the masked quote id every subsequent call needs. `createEmptyCart` does
the same job and is `@deprecated`; use `createGuestCart`.

Guest checkout is enabled on this store (`storeConfig.is_guest_checkout_enabled = true`), which
matters because the storefront's own `/customer/account/create/` and `/customer/account/login/`
routes return 404 — there is no working account path to fall back on.

## 2. Add SKUs

`addProductsToCart(cartId: String!, cartItems: [CartItemInput!]!): AddProductsToCartOutput`

```graphql
mutation {
  addProductsToCart(
    cartId: "<cart.id>"
    cartItems: [{ sku: "7K0113", quantity: 2 }]
  ) {
    cart { items { uid quantity product { sku name } } prices { grand_total { value currency } } }
    user_errors { code message }
  }
}
```

`addProductsToCart` returns `user_errors[]` rather than failing the whole request — always read
it. `addSimpleProductsToCart`, `addVirtualProductsToCart`, `addBundleProductsToCart`,
`addConfigurableProductsToCart` and `addDownloadableProductsToCart` are the older per-type
mutations; prefer the unified one.

## 3. Estimate shipping — this commits nothing

`estimateShippingMethods(input: EstimateTotalsInput!): [AvailableShippingMethod]`

```graphql
mutation {
  estimateShippingMethods(input: {
    cart_id: "<cart.id>"
    address: { country_code: US, region: { region_code: "CO" }, postcode: "80301" }
  }) { carrier_code carrier_title method_code method_title amount { value currency } }
}
```

## 4. Estimate totals — also commits nothing

`estimateTotals(input: EstimateTotalsInput!): EstimateTotalsOutput!`

```graphql
mutation {
  estimateTotals(input: {
    cart_id: "<cart.id>"
    address: { country_code: US, region: { region_code: "CO" }, postcode: "80301" }
    shipping_method: { carrier_code: "<carrier_code>", method_code: "<method_code>" }
  }) { cart { prices { subtotal_excluding_tax { value } applied_taxes { label amount { value } } grand_total { value currency } } } }
}
```

These two are the only genuine dry-run operations on the surface. Use them to answer "what would
this cost" and stop there.

## 5. Undo, if you need to

Everything before `placeOrder` is reversible, and the window is the life of the cart:

- `removeItemFromCart(input: { cart_id, cart_item_uid })`
- `updateCartItems(input: { cart_id, cart_items: [{ cart_item_uid, quantity: 0 }] })`
- `clearCart(input: { cart_uuid })`
- `setCartAsInactive(input: { cart_id })`

Once `placeOrder` converts the cart, the cart id is consumed and none of these apply.

## Country and region codes

`countries { id full_name_locale available_regions { code name } }` — or the REST twin,
`GET https://www.watchmakergenomics.com/rest/V1/directory/countries`, which is the one operation
verified live and anonymous on both transports.

## Errors

Read `user_errors[]` on cart mutations before assuming success. Transport-level failures arrive in
GraphQL's `errors[]` with an `extensions.category`. See
`errors/watchmaker-genomics-problem-types.yml`.
