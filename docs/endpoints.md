# Endpoints and models

All routes are relative to `https://argipapi.argip.com.pl/`, need `Authorization: Bearer <token>`
and use the version segment `v1` (lowercase). JSON field names are PascalCase. Names, types and
lengths are in [`openapi.json`](../openapi.json); the required/nullable rules are on this page (see
[Where `openapi.json` is not exact](#where-openapijson-is-not-exact)).

## Routes

| Method | Route | Body | 200 response |
|---|---|---|---|
| GET | `v1/Customer/Info` | none | `CustomerInfoModel` |
| GET | `v1/Categories` | none | `CategoryDto[]` |
| GET | `v1/MainCategories` | none | `CategoryDto[]` |
| GET | `v1/Categories/{id}` | none | `CategoryInfoDto` |
| GET | `v1/Products/{page}/{pagesize}/{activeonly}?quickfilter=` | none | `ProductModel[]` + header `X-Pagination` |
| GET | `v1/Products/{id}` | none | `ProductModel` |
| GET | `v1/Products/Category/{categoryid}` | none | `ProductModel[]` |
| GET | `v1/Products/BaseProduct/{baseproductid}` | none | `ProductModel[]` |
| GET | `v1/Products/Ean/{ean}` | none | `ProductModel` |
| GET | `v1/Products/YourIndex/{index}` | none | `ProductModel` |
| PUT | `v1/Products` | `MapProduct` | `ProductModel` |
| PUT | `v1/CustomTag` | `MapCustomTag` | `ProductModel` |
| GET | `v1/CustomerParts/{page}/{pagesize}` | none | `CustomerPartsModel[]` + `X-Pagination` |
| GET | `v1/CustomerParts/{productId}` | none | `CustomerPartsModel` |
| PUT | `v1/CustomerParts/` | `NewCustomerParts` | `CustomerPartsModel` |
| DELETE | `v1/CustomerParts/{productId}` | none | `CustomerPartsModel` (the deleted one) |
| GET | `v1/ShippingAddress/{page}/{pagesize}` | none | `ShippingAddressModel[]` + `X-Pagination` |
| GET | `v1/ShippingAddress/{id}` | none | `ShippingAddressModel` |
| PUT | `v1/ShippingAddress/` | `NewShippingAddress` | `ShippingAddressModel` |
| POST | `v1/Order/Calculate` | `OrderItem[]` | `CalcOrderModel` |
| POST | `v1/Order/Save` | `Order` | `OrderModel` |

- `page` starts at 0; a negative `page` gives an empty `404`. `pagesize` must be 1 to 1000, otherwise
  `400 {"message":"Exceeded page size - between 1 and 1000 items"}`.
- `activeonly` is `true` or `false`.
- URL-encode path segments and the `quickfilter` value (`A B` becomes `A%20B`). A value that contains
  `/` cannot be used in a path segment: the server does not decode `%2F`, so such a `YourIndex` or
  `Ean` lookup cannot succeed.
- An empty list (no products, no customer parts, no shipping addresses) is answered with an **empty
  `404`**, see [errors.md](errors.md).

### What the writing routes change

| Route | What it changes |
|---|---|
| `PUT v1/Products` | Your own index, name and EAN for a product (your mapping); not saved in the testing phase |
| `PUT v1/CustomTag` | Your own tag of a product; not saved in the testing phase |
| `PUT v1/CustomerParts/` | Your customer part (your own part number for a product); not saved in the testing phase |
| `DELETE v1/CustomerParts/{productId}` | Deletes your customer part; deletes nothing in the testing phase |
| `PUT v1/ShippingAddress/` | Creates a **real delivery address** at Argip; not saved in the testing phase |
| `POST v1/Order/Calculate` | Nothing (read only, although it is a `POST`) |
| `POST v1/Order/Save` | Creates a **real order**, unless `TestingPhase` is `true` |

With `TestingPhase = true` the five writes above validate exactly as in production but save nothing, and
answer `200` with a `Message` field. Details: [Testing phase](customer-data.md#testing-phase).

## Pagination header

List endpoints with `page`/`pagesize` return the header `X-Pagination`, a JSON document:

```json
{"TotalCount":16817,"TotalPages":3364,"PrevPageLink":"","NextPageLink":"https://argipapi.argip.com.pl/v1/Products/1/5/True"}
```

`PrevPageLink` and `NextPageLink` are empty strings when there is no such page. Iterate pages
`0 .. TotalPages - 1`.

## Models

Types: `int` whole numbers, `decimal` numbers with decimals, `string?` may be `null`. The API
sometimes returns empty strings instead of `null`. Field names are copied from the API, including
its spelling mistakes (marked **sic**: the field name is spelled this way in the API).

### CustomerInfoModel

| Field | Type | Meaning |
|---|---|---|
| CreditLimit | decimal | Your credit limit; never negative (0 when it is not above 0) |
| CreditAvailable | decimal | Credit still available; never negative (0 when it is not above 0), so overdrawn credit shows as 0 |
| CurrencyName | string? | Currency of your prices and orders |
| Discount | decimal | Your discount, in percent |
| ExtraDiscount | decimal | Additional general discount, in percent |
| TestingPhase | bool | `true`: orders saved with `Order/Save` are test orders and are not processed; the other writes are validated but not saved ([customer-data.md](customer-data.md#testing-phase)) |
| Language | string? | Account language, `PL` or `EN` (product names, tags and category names follow it; a category name falls back to Polish when the English one is empty, `BreadcrumbsTrail` does not) |

### CategoryDto

| Field | Type | Meaning |
|---|---|---|
| CategoryId | int | Category ID |
| ParentCategoryId | int | ID of the parent category; `0` means a top-level category |
| Name | string? | Name in your account language |

### CategoryInfoDto

| Field | Type | Meaning |
|---|---|---|
| CategoryId | int | Category ID |
| BreadcrumbsTrail | string? | Readable path to the category, ordered from the top-level category down to the category itself (root to leaf), joined with ` » ` |
| PathElements | CategoryDto[]? | The categories on the path, ordered from the category itself up to the top-level one (leaf to root) |

### ProductModel

| Field | Type | Meaning |
|---|---|---|
| ProductId | int | Product ID; the key used in orders |
| BaseProductId | int | Groups the packing units of one product (see `Products/BaseProduct/{id}`) |
| Index | string? | Argip product index |
| ProductFullName | string? | Product name in your account language |
| EanBarcode | string? | EAN of a single package |
| MultiPackEanBarcode | string? | EAN of a multi-pack |
| SinglePackQuantityInPieces | int | Pieces in one package. Orders are rounded up to full packages |
| MultiPackQuantityInPieces | int | Number of single packages in a multi-pack (**despite the name, not pieces**). Pieces in a multi-pack = `MultiPackQuantityInPieces * SinglePackQuantityInPieces` |
| QuantityLimitLevel1 | int | From this quantity (pieces) `YourPriceLevel1` applies; 0 means no such level |
| QuantityLimitLevel2 | int | From this quantity (pieces) `YourPriceLevel2` applies; 0 means no such level |
| PictureUrl | string? | Product picture URL; empty string when there is none |
| MainPrice | decimal | Argip list price per 100 pieces in your currency, before your discount |
| YourMainPrice | decimal | Your price per 100 pieces (after discounts) below the first quantity limit |
| YourPriceLevel1 | decimal | Your price per 100 pieces from `QuantityLimitLevel1`; 0 means no level |
| YourPriceLevel2 | decimal | Your price per 100 pieces from `QuantityLimitLevel2`; 0 means no level |
| CurrencyId | int | Currency ID (for example 1 = PLN, 2 = EUR); use `CurrencyName` for display |
| CurrencyName | string? | Currency of the prices |
| PiecesInStock | int | Pieces in stock |
| OtherPossibleQuantity | int | Quantity available for repacking or in other packages, reported separately from stock. `Order/Calculate` and `Order/Save` look only at `PiecesInStock` |
| NearestDilivery | string? | (**sic**: "Dilivery") Currently always `"-"`; do not rely on it |
| YourIndex | string? | Your index, set with `PUT Products`; empty string when not set, `null` after a `PUT Products` that left it out |
| YourProductFullName | string? | Your product name, set with `PUT Products`; empty string when not set, `null` after a `PUT Products` that left it out |
| YourEanBarcode | string? | Your EAN, set with `PUT Products`; empty string when not set, `null` after a `PUT Products` that left it out |
| IsActive | bool | `false` for a product that is still listed but cannot be ordered |
| BoxWeight | decimal | Weight of one package (not related to `ItemWeight` by a simple multiplication; use `ItemWeight`/`TotalWeight` from `Order/Calculate` for shipping weight) |
| TaxRate | decimal | VAT rate in percent |
| CountryOfOrigin | string? | Country of origin; empty string when unknown |
| IsDeviation | bool | The product has a deviation |
| Deviation | string? | Description of the deviation |
| Tags | TagModel[]? | Product attributes in your account language |
| CustomTags | CustomTagModel[]? | Your own tags (set with `PUT CustomTag`) |
| CategoryMapping | int[]? | IDs of the categories the product belongs to |
| Message | string? | Set only in the response of `PUT Products` and `PUT CustomTag` for an account in the testing phase: the data was validated but not saved. Absent from every other response |

### TagModel and CustomTagModel

| Model | Field | Type | Meaning |
|---|---|---|---|
| TagModel | Id | int | Tag type ID. Known values include 1 name/description, 2 standard, 3 class/material, 4 coating, 5 attribute, 6 size, 7 other, 10 pre-applied; the list may grow |
| TagModel | Value | string? | Tag value |
| CustomTagModel | Type | int | Your tag type (a number you choose) |
| CustomTagModel | Value | string? | Your tag value |

### MapProduct (request of `PUT v1/Products`)

| Field | Type | Meaning |
|---|---|---|
| ProductId | int | Product to map (must be active and on offer) |
| Index | string? | Your index for the product; max 50 characters |
| ProductFullName | string? | Your name for the product; max 250 characters |
| EanBarcode | string? | Your EAN for the product; max 50 characters |

`Index`, `ProductFullName` and `EanBarcode` are all overwritten with what you send: a field you
leave out is cleared. A value over its limit gives `400` with a validation object
(`{"Index":["Index should have max 50 characters"]}`), not a `{"message":...}` body.

### MapCustomTag (request of `PUT v1/CustomTag`)

| Field | Type | Meaning |
|---|---|---|
| ProductId | int | Product (must be active and on offer) |
| TagType | int | Your tag type; one value per type and product |
| TagValue | string | The value; required, not empty, max 50 characters; replaces the existing value of this type. A tag cannot be cleared with an empty value |

A missing or too long `TagValue` gives `400` with a validation object such as
`{"TagValue":["Tag value should have max 50 characters"]}`.

### CustomerPartsModel

| Field | Type | Meaning |
|---|---|---|
| CustomerPartsId | int | ID of the customer part |
| ProductId | int | Product ID |
| Part | string? | Your part number for the product |
| Descrption | string? | (**sic**: "Descrption") Your description |
| Message | string? | Set only in the response of `PUT`/`DELETE CustomerParts` for an account in the testing phase: the data was validated but not saved. Absent from every other response |

### NewCustomerParts (request of `PUT v1/CustomerParts/`)

| Field | Type | Meaning |
|---|---|---|
| Part | string | Your part number; must not be empty or only spaces; max 50 characters (longer is cut off) |
| Description | string? | Your description (spelled correctly in the request); max 255 characters (longer is cut off); send `""`, not `null`, when you have none |
| ProductId | int | Product ID |

### ShippingAddressModel

| Field | Type | Meaning |
|---|---|---|
| ShippingAddressId | int | Address ID, used in `Order.ShippingAddressId` |
| FiendlyName | string? | (**sic**: "Fiendly") Short name of the address |
| PostalCode | string? | Postal code |
| Street | string? | Street |
| BuildingNumber | string? | Building number |
| City | string? | City |
| CountryCode | string? | Country code (ISO 3166-1 alpha-2) |
| ContactPhone | string? | Contact phone |
| ContactPerson | string? | Contact person |
| ReceivingCompanyName | string? | Receiving company |
| Message | string? | Set only in the response of `PUT ShippingAddress` for an account in the testing phase: the data was validated but not saved (`ShippingAddressId` is then `0`). Absent from every other response |

### NewShippingAddress (request of `PUT v1/ShippingAddress/`)

| Field | Type | Meaning |
|---|---|---|
| FiendlyName | string | (**sic**) Short name; may be empty; max 50 |
| PostalCode | string | Postal code (`NN-NNN` for Poland); max 10 |
| Street | string | Street; max 50 |
| BuildingNumber | string | Building number; max 10 |
| City | string | City; max 30 |
| CountryCode | string | ISO country code, for example `PL`; max 2 |
| ContactPhone | string | Contact phone; max 20. For Poland 9 digits; every character except digits and `+` is removed, then a `0048` prefix and leading zeros, and `+48` is added when missing, so `123456789` is accepted |
| ContactPerson | string | Contact person; max 50 |
| ReceivingCompanyName | string | **Required**: not empty and not only spaces; max 50 |
| Email | string | **Required** contact e-mail; max 250. The address is validated, and a missing or `null` value makes the call fail with an empty `404` |

### OrderItem (request item of `Order/Calculate`, part of `Order`)

| Field | Type | Meaning |
|---|---|---|
| ProductId | int | Product ID |
| QuantityInPieces | int | Quantity in pieces. `0` or less is **not** rejected: it becomes one full package without a warning, so filter such rows out first |

### Order (request of `POST v1/Order/Save`)

| Field | Type | Meaning |
|---|---|---|
| ShippingAddressId | int | One of your active shipping addresses |
| CustomerOrderNumber | string? | Your own order number, stored with the order; **max 30 characters** (a longer value makes Save fail with an empty `404` and nothing is saved) |
| ExecutionDate | date-time | Requested execution date, for example `"2026-11-12T00:00:00"`; not earlier than today in Poland (Europe/Warsaw) |
| Comments | string? | Comments for the order |
| Items | OrderItem[] | 1 to 500 items. Missing or `null` gives an empty `404` |

### WarningMessage

| Field | Type | Meaning |
|---|---|---|
| Type | int | 1 to 5, see [orders.md](orders.md) |
| ProductId | int? | The product concerned |
| Comment | string? | English text describing what happened |

### OrderModelItem

| Field | Type | Meaning |
|---|---|---|
| Product | ProductModel? | The product |
| Price | decimal | Price per 100 pieces applied (after quantity levels) |
| TaxRate | decimal | VAT rate applied, in percent; 0 when your account is charged no VAT |
| PiecesOrdered | int | Quantity you asked for (duplicates of one product added up) |
| PiecesReceived | int | Quantity actually accepted after rounding and stock limits |
| ItemNetValue | decimal | Net value of the item: `Price * PiecesReceived / 100`, rounded to 2 decimals |
| ItemWeight | decimal | Weight of the item (use this, not `BoxWeight`, for shipping weight) |

### CalcOrderModel (response of `Order/Calculate`)

| Field | Type | Meaning |
|---|---|---|
| TotalWeight | decimal | Total weight |
| TotalNetValue | decimal | Total net value |
| TotalGrossValue | decimal | Total gross value (VAT per rate, see [quantities-and-prices.md](quantities-and-prices.md)) |
| TaxValue | decimal | Gross minus net |
| CurrencyName | string? | Currency |
| Items | OrderModelItem[]? | Accepted items |
| WarningMessages | WarningMessage[]? | Warnings, see [orders.md](orders.md) |

### OrderModel (response of `Order/Save`)

The same totals, `CurrencyName`, `Items` and `WarningMessages` as `CalcOrderModel`, plus
`ShippingAddress` (`ShippingAddressModel?`), and `CustomerOrderNumber`, `ExecutionDate` (date only,
time 00:00:00) and `Comments` echoed from the request.

## Where `openapi.json` is not exact

The required and nullable flags in [`openapi.json`](../openapi.json) come from the API's annotations
and do not describe real behaviour everywhere. When they differ from this guide, this guide wins:

- `NewShippingAddress.Email` is marked optional, but it is **required** (missing or `null`: empty `404`).
- `Order.Items` is marked optional, but it is **required** (missing or `null`: empty `404`).
- `NewShippingAddress.ReceivingCompanyName` is marked optional, but it is **required** (empty: `404` `friendly and company name couldn't be empty`).
- Strings marked nullable in write bodies (`Description`): send `""`, never `null`.
- Response strings are declared as non-nullable `string`, but any of them can be `null`.
- Length limits for `PUT ShippingAddress` and `PUT CustomerParts` are in the schema, but the API does
  not reject longer strings: it cuts them off; see [customer-data.md](customer-data.md).
