# Products and categories

All examples use illustrative values, not real catalogue data.

## What the API returns

Only products that are **on offer** are visible. Of these, some are `IsActive = false`: they are
listed but cannot be ordered (`Order/Calculate` and `Order/Save` skip them with warning type 5).
A product that is not on offer behaves as if it did not exist (`404`).

## Listing the catalogue

```http
GET /v1/Products/0/2/true HTTP/1.1
Authorization: Bearer <token>
```

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Pagination: {"TotalCount":16817,"TotalPages":8409,"PrevPageLink":"","NextPageLink":"https://argipapi.argip.com.pl/v1/Products/1/2/True"}
```

```json
[
  {
    "ProductId": 1001,
    "BaseProductId": 1001,
    "Index": "ABC-0001",
    "ProductFullName": "Hex head screw DIN 931 M8x80",
    "EanBarcode": "5901234567890",
    "MultiPackEanBarcode": null,
    "SinglePackQuantityInPieces": 200,
    "MultiPackQuantityInPieces": 0,
    "QuantityLimitLevel1": 1000,
    "QuantityLimitLevel2": 5000,
    "PictureUrl": "",
    "MainPrice": 14.50,
    "YourMainPrice": 12.50,
    "YourPriceLevel1": 11.90,
    "YourPriceLevel2": 11.20,
    "CurrencyId": 1,
    "CurrencyName": "PLN",
    "PiecesInStock": 4000,
    "OtherPossibleQuantity": 0,
    "NearestDilivery": "-",
    "YourIndex": "",
    "YourProductFullName": "",
    "YourEanBarcode": "",
    "IsActive": true,
    "BoxWeight": 1.2,
    "TaxRate": 23,
    "CountryOfOrigin": "DE",
    "IsDeviation": false,
    "Deviation": null,
    "Tags": [ { "Id": 5, "Value": "Zinc" } ],
    "CustomTags": [],
    "CategoryMapping": [ 12, 40 ]
  }
]
```

- `page` starts at 0 (a negative `page` gives an empty `404`), `pagesize` is 1 to 1000. Use 1000 for bulk reads.
- `activeonly=true` returns active products only; `false` also returns inactive ones.
- Products are sorted by `ProductId`, so paging is stable between calls unless the catalogue changes.
- Read `TotalPages` from `X-Pagination` and request pages `0 .. TotalPages - 1`. A page past the
  end returns an empty array.
- If nothing matches (for example a `quickfilter` without results) the response is `404` with an
  **empty body**. That is "no results" here, not an error; but see [errors.md](errors.md) for why an
  empty `404` is ambiguous.

### Filtering by name

`quickfilter` searches product names in your account language. URL-encode the value:

```http
GET /v1/Products/0/20/true?quickfilter=DIN%20931%20M8x80 HTTP/1.1
```

## Looking up a single product

| Route | Notes |
|---|---|
| `GET v1/Products/{id}` | By `ProductId`. `404` `{"message":"Product with id = 1001 not found"}` |
| `GET v1/Products/Ean/{ean}` | Searches the EAN of a single package first, then the multi-pack EAN. `404` `{"message":"Product with Ean = 5901234567890 not found"}` |
| `GET v1/Products/YourIndex/{index}` | By the index you set with `PUT Products`. `404` `{"message":"No custom mappings defined"}` if you never mapped anything, `{"message":"Product with Index X not found"}` otherwise |
| `GET v1/Products/Category/{categoryid}` | Products of one category. `404` `{"message":"Products with categoryId = N not found"}` |
| `GET v1/Products/BaseProduct/{baseproductid}` | Other packing units of the same product. `404` `{"message":"Products with BaseProductId = N not found"}` |

Single-product lookups can return an inactive product (`IsActive = false`); check the flag before
you order it. Category and base product lookups return only active products on offer, as plain
arrays without `X-Pagination`. `Products/Category/{id}` uses only the products mapped directly to
that category (not its subcategories) and gives `200 []` when the category has mappings but no
active product.

Always URL-encode path values: `GET v1/Products/YourIndex/A%20B` for the index `A B`.

A value containing `/` cannot be looked up this way: the server does not decode `%2F` in a path
segment, so the request ends in an **empty** `404` (not the `Product with Index ... not found` message). Do not
use `/` in your own indexes, or find such a product by another route (`ProductId`, or `Ean` when the EAN has no `/`).

## Your own index, name and EAN

`PUT v1/Products` stores your own data for a product, so you can find it later with
`Products/YourIndex/{index}` and read it back in `YourIndex`, `YourProductFullName` and
`YourEanBarcode`:

```http
PUT /v1/Products HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{ "ProductId": 1001, "Index": "SCREW-M8-80", "ProductFullName": "M8x80 screw", "EanBarcode": "" }
```

The response is the updated `ProductModel`. **All three fields are overwritten**: `EanBarcode` above
is cleared. Read the product first and send back the values you want to keep. The product must be
active and on offer, otherwise `404` `{"message":"Product with id = 1001 not found or is not active"}`.

`PUT v1/CustomTag` works the same way for tags: body `{"ProductId": 1001, "TagType": 7, "TagValue":
"shelf A3"}`; you choose the numeric `TagType`. One value is kept per type and product, a new
`TagValue` replaces the old. The tags come back in `CustomTags`. `TagValue` is required, not empty and
max 50 characters, so a tag cannot be cleared with an empty value. A violation gives `400` with a
validation object, for example `{"TagValue":["Tag value should have max 50 characters"]}`.

`PUT v1/Products` has its own limits: `Index` 50, `ProductFullName` 250, `EanBarcode` 50 characters.
Both `PUT`s run their checks in this order: account (`400` `Unknown client`), product (`404`), field
validation (`400`). So a too long value for an unknown product gives `404`, not `400`.

An unknown account gives `400` with the text `Unknown client`.

In the testing phase both `PUT`s run the same checks but save nothing: the response is the product
built from the data you sent, with a `Message`, and reading the product afterwards shows the old
`YourIndex` and `CustomTags` ([customer-data.md](customer-data.md#testing-phase)).

## Categories

- `GET v1/Categories` returns every category; `ParentCategoryId` links a category to its parent, so
  the list is the whole tree. `ParentCategoryId = 0` marks a top-level category.
- `GET v1/MainCategories` returns the top-level categories.
- `GET v1/Categories/{id}` returns `CategoryInfoDto` with `BreadcrumbsTrail` (root to leaf, joined with
  ` » `) and `PathElements` (leaf to root).
  Unknown id: `404` `{"message":"Category with id = 99999 not found"}`.
- A product lists its categories in `CategoryMapping` (IDs).
- `Products/{id}`, `Products/Category/{id}` and `Products/BaseProduct/{id}` answer `400`
  `{"message":"User <client_id> not attached to real client"}` when the account is not set up for API use.

## Pitfalls

- `NearestDilivery` is always `"-"`.
- `PictureUrl` is an empty string when the product has no picture.
- `YourIndex` etc. are empty strings (not `null`) until you map the product. After a `PUT Products` that
  left a field out, that field is `null`.
- Prices are per 100 pieces; see [quantities-and-prices.md](quantities-and-prices.md).
- `PiecesInStock` changes continuously; treat a catalogue snapshot as old data and rely on
  `Order/Calculate` for the final check.
