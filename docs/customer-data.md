# Customer data

Account info, shipping addresses and customer parts. All data belongs to the account that owns the
token; you never see another customer's data.

## Account info

```http
GET /v1/Customer/Info HTTP/1.1
Authorization: Bearer <token>
```

```json
{
  "CreditLimit": 50000.00,
  "CreditAvailable": 41250.50,
  "CurrencyName": "PLN",
  "Discount": 12.0,
  "ExtraDiscount": 0.0,
  "TestingPhase": true,
  "Language": "PL"
}
```

- `TestingPhase = true` means `Order/Save` stores test orders that are not processed, and the other
  writes save nothing ([Testing phase](#testing-phase), [orders.md](orders.md)). Check it before any test.
- `Language` decides the language of product names, tags and category names.
- This is the cheapest call to verify that your token and credentials work. An unknown account
  gets an empty `404`.

## Testing phase

When `Customer/Info` returns `TestingPhase = true`, these writes run exactly the same validation as in
production (same `400` and `404` answers) but **save nothing**:

- `PUT v1/CustomTag`
- `PUT v1/Products` (your index, name and EAN)
- `PUT v1/ShippingAddress/`
- `PUT v1/CustomerParts/`
- `DELETE v1/CustomerParts/{productId}` (nothing is deleted)

The answer is `200` with the same model as in production, built from the data you sent, plus a `Message`:

```json
{ "CustomerPartsId": 0, "ProductId": 1001, "Part": "MY-001", "Descrption": "My screw",
  "Message": "Testing phase: the data is valid and would be saved in the production phase." }
```

- A new shipping address has `ShippingAddressId = 0`; a new customer part has `CustomerPartsId = 0`
  (an existing part of that product keeps its id).
- `Message` is present only in these responses; everywhere else the property is absent from the JSON
  (not `null`).
- Reading the data back (`GET v1/ShippingAddress/0`, `GET v1/CustomerParts/{productId}`, `CustomTags`
  and `YourIndex` of a product) shows the **old** state, or `404`. Skip read-back checks in tests, or
  expect the old state.
- After your account moves to production, send this data again: nothing from the testing phase was stored.
- `Order/Save` is unchanged: in the testing phase it stores a test order that is never processed.

## Shipping addresses

`GET v1/ShippingAddress/{page}/{pagesize}` lists your **active** addresses sorted by `FiendlyName`
(**sic**), with `X-Pagination` like the product list. No addresses: empty `404`.
`GET v1/ShippingAddress/{id}` returns one; unknown or inactive:
`404` `{"message":"Shipping address with id = 4321 not found"}`.

```json
{
  "ShippingAddressId": 4321,
  "FiendlyName": "Warehouse Gdansk",
  "PostalCode": "80-001",
  "Street": "Example Street",
  "BuildingNumber": "10",
  "City": "Gdansk",
  "CountryCode": "PL",
  "ContactPhone": "+48123456789",
  "ContactPerson": "Jan Kowalski",
  "ReceivingCompanyName": "Example Ltd"
}
```

`ShippingAddressId` is what you put in `Order.ShippingAddressId`.

### Creating an address

`PUT v1/ShippingAddress/` **creates a real delivery address at Argip**, unless your account is in the
testing phase: then it is validated but not saved ([Testing phase](#testing-phase)). It returns the new
`ShippingAddressModel` (`ShippingAddressId = 0` and a `Message` in the testing phase).

```http
PUT /v1/ShippingAddress/ HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{
  "FiendlyName": "Warehouse Gdansk", "PostalCode": "80-001", "Street": "Example Street",
  "BuildingNumber": "10", "City": "Gdansk", "CountryCode": "PL", "ContactPhone": "+48123456789",
  "ContactPerson": "Jan Kowalski", "ReceivingCompanyName": "Example Ltd", "Email": "jan@example.com"
}
```

Rejections are `404` with a `message` (not `400`):

| `message` | Cause |
|---|---|
| `country code is incorrect` | `CountryCode` is not a valid ISO 3166-1 alpha-2 code (case is ignored) |
| `email is incorrect` | `Email` does not look like an address (a missing `Email` gives an empty `404`) |
| `phone number is incorrect` | Poland only: the phone is not `+48` plus 9 digits after cleaning (every character except digits and `+` is removed, then a `0048` prefix and leading zeros; `+48` is added when missing) |
| `postal code is incorrect` | Poland only: not `NN-NNN` (5 digits without the dash are accepted and fixed) |
| `friendly and company name couldn't be empty` | `ReceivingCompanyName` is empty or only spaces (it is required; `FiendlyName` may be empty) |
| `postal code couldn't be empty`, `street name couldn't be empty`, `building number couldn't be empty` | The field is empty or only spaces (for `Street` only the first 30 characters count) |
| `incorret contractor id` | (**sic**) Account problem; contact Argip |
| `error code: -N` | Another failure; contact Argip |
| `Shipping address with id = N not found` | The address was created but could not be read back |

A `null` in a string field (for example `CountryCode`) fails with an empty `404`; send `""`, not `null`,
for values that may be empty (`FiendlyName`, `City`), and always send `Email` and `ReceivingCompanyName`.
An empty `City` is accepted.

Length limits: `FiendlyName` 50, `PostalCode` 10, `Street` 50, `BuildingNumber` 10, `City` 30,
`CountryCode` 2, `ContactPhone` 20, `ContactPerson` 50, `ReceivingCompanyName` 50, `Email` 250. A longer
value is silently cut off to the limit, not rejected: check lengths before sending.

## Customer parts

Your own part numbers for Argip products. One customer part per product.

| Call | Notes |
|---|---|
| `GET v1/CustomerParts/{page}/{pagesize}` | Paged like products; empty `404` when you have none |
| `GET v1/CustomerParts/{productId}` | `404` `{"message":"Customer part with product id = 1001 not found"}` |
| `PUT v1/CustomerParts/` | Creates the part for the product or updates the existing one. Body `{"Part":"MY-001","Description":"My screw","ProductId":1001}`. Saves nothing in the testing phase |
| `DELETE v1/CustomerParts/{productId}` | Deletes it and returns the deleted part; unknown: same `404` as GET. Deletes nothing in the testing phase |

Response shape:

```json
{ "CustomerPartsId": 77, "ProductId": 1001, "Part": "MY-001", "Descrption": "My screw" }
```

Note the response field is spelled `Descrption` (**sic**) while the request field is `Description`.

Length limits: `Part` 50, `Description` 255; a longer value is silently cut off, not rejected.

PUT errors, all `404` with a `message`: `incorret product id` (**sic**; `ProductId` is not in the
catalogue; an inactive product is accepted), `part couldn't be empty` (empty or only spaces),
`error code: -N`, `Customer parts with product id = N not found`. A `null` `Description` gives an empty
`404`: send `""`. The same checks run in the testing phase.

This is different from `PUT v1/Products`, which stores your index, name and EAN as a product
mapping ([products.md](products.md)).
