# Orders

Two calls: `POST v1/Order/Calculate` checks and prices a basket without saving anything,
`POST v1/Order/Save` does the same checks and then **places the order**.

Always send `Content-Type: application/json` (without it the API answers with an empty `415`).

## Calculate

```http
POST /v1/Order/Calculate HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

[ { "ProductId": 1001, "QuantityInPieces": 250 }, { "ProductId": 999999, "QuantityInPieces": 10 } ]
```

```json
{
  "TotalWeight": 2.4,
  "TotalNetValue": 50.00,
  "TotalGrossValue": 61.50,
  "TaxValue": 11.50,
  "CurrencyName": "PLN",
  "Items": [
    {
      "Product": { "ProductId": 1001, "SinglePackQuantityInPieces": 200, "PiecesInStock": 4000, "...": "..." },
      "Price": 12.50,
      "TaxRate": 23,
      "PiecesOrdered": 250,
      "PiecesReceived": 400,
      "ItemNetValue": 50.00,
      "ItemWeight": 2.4
    }
  ],
  "WarningMessages": [
    { "Type": 3, "ProductId": 1001, "Comment": "Revised quantity from 250 to 400. Only full packages can be sold!" },
    { "Type": 1, "ProductId": 999999, "Comment": "Product with id = 999999 not found" }
  ]
}
```

(Values are illustrative; `Product` is a full `ProductModel`.)

- Nothing is saved. Use it to check a basket or to prepare an offer.
- Items with the same `ProductId` are added up.
- Limits: at least 1 and at most **500** items. Otherwise `400`
  `{"message":"Add some Items to Your Order"}` or
  `{"message":"A maximum of 500 items can be calculated simultaneously."}`.
  A body that is not a JSON array gives `400` `{"message":"Bad Items structure"}`.
- The product, quantity and warning logic is the same as in Save, except for stock shortages (below).

## Save

```http
POST /v1/Order/Save HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{
  "ShippingAddressId": 4321,
  "CustomerOrderNumber": "PO-2026-0042",
  "ExecutionDate": "2026-11-12T00:00:00",
  "Comments": "Deliver before noon",
  "Items": [ { "ProductId": 1001, "QuantityInPieces": 250 } ]
}
```

The response is an `OrderModel`: the same totals, `Items` and `WarningMessages` as in Calculate,
plus `ShippingAddress`, `CustomerOrderNumber`, `ExecutionDate` and `Comments`.

**Save places a real order** that Argip processes, unless your account is in the testing phase
(`TestingPhase = true` in `GET v1/Customer/Info`). Test orders are stored but not processed, so
use such an account while developing. Ask Argip to put your account into the testing phase. The other
writes behave differently there too: see [Testing phase](customer-data.md#testing-phase).

Checks, in this order:

| Condition | Response |
|---|---|
| Body not a JSON object | `400` `{"message":"Bad Order structure"}` |
| `Items` empty | `400` `{"message":"Add some Items to Your Order"}` (`Items` missing or `null`: empty `404`) |
| More than 500 items | `400` `{"message":"A maximum of 500 items can be saved to one order."}` |
| `ExecutionDate` before today | `400` `{"message":"Bad execution date. Min <today>"}` (the date text depends on the server's regional settings; do not parse it) |
| `ShippingAddressId` is not one of your active addresses | `404` `{"message":"Shipping address with id = 4321 not found"}` |

Only the date part of `ExecutionDate` counts; the response returns it with time 00:00:00. The server
compares it with today's date in Poland (Europe/Warsaw), so send that date or a later one as
`YYYY-MM-DDT00:00:00` without an offset. Compute "today" in the `Europe/Warsaw` time zone, not in
your own: a customer west of Poland who sends its local date late in the day can get
`400 Bad execution date`.

### 200 does not mean saved

The order is saved only if **at least one item remains** after the checks. If every item is
rejected, the response is still `200 OK` with empty `Items`, and **nothing is saved**. After every
Save:

1. Check that `Items` is not empty.
2. Compare `PiecesReceived` with `PiecesOrdered` for every item.
3. Read every `WarningMessages` entry and show it to the user.

Save has no idempotency key. If a Save call times out or returns an empty `404`, the outcome is
**unknown**: the order may or may not exist. **Never resend it automatically**, a retry can create a
duplicate order. Check with Argip first (the "retry later" advice for an empty `404` applies to reads
only). Use a unique `CustomerOrderNumber` per order so you and Argip can recognise duplicates.

`CustomerOrderNumber` is at most **30 characters**. A longer value makes Save fail with an empty
`404` and nothing is saved (a GUID-based number of 36 characters does not fit). Keep it short and
check the length before sending.

## Warning types

| Type | Where | `Comment` text (exact; `N`, `X`, `Y` are numbers) | What happened |
|---|---|---|---|
| 1 | Calculate, Save | `Product with id = N not found` | Unknown product. The item is skipped |
| 5 | Calculate, Save | `Product with id = N is not active or is not currently on offer` | Inactive product, not on offer, or without a valid package size. The item is skipped |
| 3 | Calculate, Save | `Revised quantity from X to Y. Only full packages can be sold!` | Quantity larger than one package was rounded up to full packages, and Y is within stock |
| 2 | Calculate | `Quantity unreachable. Correct this before saving order. Actual stock level - N pcs.` | The (rounded) quantity is more than the stock, or the stock is 0 or less. In Calculate the item **stays** in the calculation with the rounded quantity |
| 2 | Save | `Exceeded stock level. Quantity reduced from X to Y.` | The quantity was reduced to the stock level (Y), which may not be a whole number of packages. `X` is the quantity before the reduction |
| 4 | Save only | `Out of stock. Product with id = N removed from calculation/order.` | The stock is 0 or less; the item was removed from the order |

Notes:

- Quantity up to one package becomes one package without any warning. This includes `0` and negative
  quantities: `QuantityInPieces` of 0 or less is not rejected, it orders one full package with no
  warning (duplicates are summed first, so `[100, -100]` also gives one package). Filter out
  non-positive quantities before Calculate and Save.
- In Calculate a product with no stock gets type 2 and stays in the totals; in Save the same product
  gets type 4 and is removed. Never use Calculate totals as a promise of availability.
- `WarningMessage.ProductId` is the product concerned.
- Show `Comment` to people as is (it is English), and branch your code on `Type`, not on the text.

## Testing safely

1. Get an account in the testing phase from Argip; confirm with `GET v1/Customer/Info`
   (`TestingPhase = true`).
2. Call `Order/Calculate`, then `Order/Save`, and check `Items`/`WarningMessages`.
3. Use `CustomerOrderNumber` values that mark test orders (for example prefix `TEST-`).
4. Never run your integration tests with a production account that is not in the testing phase.
5. In the testing phase `PUT CustomTag`, `PUT Products`, `PUT ShippingAddress`, `PUT CustomerParts` and
   `DELETE CustomerParts` save nothing, so a read-back check must be skipped or must expect the old
   state ([customer-data.md](customer-data.md#testing-phase)).

## Pitfalls

- Send pieces, not packages.
- A saved order is not editable through the API; there is no cancel call.
- `ShippingAddressId` comes from `GET v1/ShippingAddress/...` ([customer-data.md](customer-data.md)).
- Argip may send an order confirmation e-mail.
