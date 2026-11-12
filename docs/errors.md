# Errors

## Status codes

| Status | When | Body |
|---|---|---|
| `200` | Success. For `Order/Save` this does not mean the order was saved ([orders.md](orders.md)) | JSON |
| `400` | Bad `pagesize`; bad order input (empty, too many items, bad date, bad body); the account is not set up for API use; invalid `PUT Products` / `PUT CustomTag` input | `{"message":"..."}`, a validation object `{"<Field>":["<text>"]}` (the two `PUT`s) or empty |
| `401` | Missing, invalid or expired token | Empty, header `WWW-Authenticate` starting with `Bearer` |
| `404` | Business "not found" or rejected input (a `message`); no results in a list; **and** unhandled server failures; **and** a wrong HTTP method | `{"message":"..."}` or **empty** |
| `415` | A body endpoint was called without a supported `Content-Type` (send `application/json`) | Empty |

There is no `500`: an unhandled failure on the server is returned as an **empty `404`**.

## The empty 404

An empty `404` can mean:

1. A list with no results (`Products` with a `quickfilter` that matches nothing, no shipping
   addresses, no customer parts), or a negative `page`.
2. A server-side failure, or an account that is not set up for API use (`Customer/Info`).
3. A wrong HTTP method on a known route (for example `GET` on `v1/Order/Save`), or a wrong route.
4. A wrong version segment: only lowercase `v1` works (`V1` gives an empty `404`).
5. A request body that makes the server fail, for example `null` in a required field.
6. A path value containing `/` (`%2F`), for example in `Products/YourIndex/{index}` or
   `Products/Ean/{ean}`: the server does not decode it, so the lookup cannot succeed.

How to handle it:

- For single-resource lookups the API normally answers `404` **with** a `message`. An empty `404`
  there suggests a server error.
- Log the request (method, route, no secrets) and the response status.
- For **reads** (`GET`, and `POST Order/Calculate`, which saves nothing): retry later with backoff (a few
  attempts), then report it to Argip with the time and route.
- For **writes** (`Order/Save`, `PUT`, `DELETE`) an empty `404` or a timeout means the outcome is
  **unknown**. Do not resend automatically: check the state first (for `Order/Save` check with Argip),
  or you may create a duplicate. Do not treat it as proof that an order was not saved.
- An empty `404` on `Order/Save` can also come from a `CustomerOrderNumber` longer than 30 characters
  (nothing is saved), or a missing/`null` `Items`.

## Business errors with a message

The body is `{"message":"..."}`. Common texts:

| Where | Status | `message` |
|---|---|---|
| Lists | 400 | `Exceeded page size - between 1 and 1000 items` |
| `Products/{id}` | 404 | `Product with id = N not found` |
| `Products/Ean/{ean}` | 404 | `Product with Ean = X not found` |
| `Products/YourIndex/{index}` | 404 | `No custom mappings defined`; `Product with Index X not found` |
| `Products/Category/{id}` | 404 | `Products with categoryId = N not found` |
| `Products/BaseProduct/{id}` | 404 | `Products with BaseProductId = N not found` |
| `PUT Products`, `PUT CustomTag` | 404 | `Product with id = N not found or is not active` |
| `PUT Products`, `PUT CustomTag` | 400 | validation object, for example `{"TagValue":["Tag value should have max 50 characters"]}`; limits: `Index` 50, `ProductFullName` 250, `EanBarcode` 50, `TagValue` required and max 50 |
| `Products/{id}`, `Products/Category/{id}`, `Products/BaseProduct/{id}` | 400 | `User <client_id> not attached to real client` |
| `Categories/{id}` | 404 | `Category with id = N not found` |
| `Order/Calculate` | 400 | `Bad Items structure`; `Add some Items to Your Order`; `A maximum of 500 items can be calculated simultaneously.` |
| `Order/Save` | 400 | `Bad Order structure`; `Add some Items to Your Order`; `A maximum of 500 items can be saved to one order.`; `Bad execution date. Min <date>` |
| `Order/Save` | 404 | `Shipping address with id = N not found` |
| `ShippingAddress/{id}`, `CustomerParts/{id}` | 404 | `Shipping address with id = N not found`; `Customer part with product id = N not found` |
| `PUT ShippingAddress`, `PUT CustomerParts` | 404 | validation texts, see [customer-data.md](customer-data.md). The same checks run in the testing phase ([customer-data.md](customer-data.md#testing-phase)) |

Some `400` answers have an empty body or a plain text (`Unknown client` for `PUT Products` and
`PUT CustomTag`) when the account is not set up for API use: contact Argip.

Note that validation of `PUT` addresses and customer parts also uses `404`, not `400`.

## Token server errors

From `https://identityserver.argip.com.pl/connect/token`: `400` with `{"error":"invalid_client"}`,
`invalid_request`, `unsupported_grant_type`, `invalid_scope`. See [authentication.md](authentication.md).

## Handling errors in code

1. Read status first; parse `{"message"}` only when the body is not empty.
2. Keep both the status and the raw body in your exception, so a caller can see an empty `404`.
3. `401`: refresh the token once and retry once.
4. `400` and `404` with a `message`: do not retry; show or log the message.
5. Empty `404`, network errors and timeouts on **reads**: retry a few times with growing delays.
6. Never automatically retry `Order/Save`, `PUT` and `DELETE` after an empty `404` or a timeout: the
   outcome is unknown and a retry can duplicate the order. Check the state first
   ([orders.md](orders.md)). The `401` retry (step 3) is safe because a `401` is rejected before
   anything is processed.
