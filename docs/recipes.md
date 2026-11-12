# Recipes

Three common integrations as language-independent steps. For code in your language start from
the example repositories ([examples.md](../examples.md)).

All recipes assume the token handling from [authentication.md](authentication.md): one cached
token, refreshed 5 minutes before expiry, one retry after `401`.

## 1. Nightly catalogue sync with stock and prices

Goal: keep your own database in step with Argip's offer, including `PiecesInStock` and your prices.

1. Get a token (or reuse the cached one).
2. `GET v1/Products/0/1000/true`. Read `TotalPages` from the `X-Pagination` header of the response.
   - `404` with an empty body means the catalogue came back empty: treat it as an error for a
     nightly sync, do not delete your data.
3. Process the array of `ProductModel`.
4. Repeat for pages `1 .. TotalPages - 1` with the same `pagesize` (you can also follow
   `NextPageLink` until it is empty).
5. Collect everything in memory or a staging table. Do not touch the live data yet.
6. Compare with your data by `ProductId` (the stable key):
   - new `ProductId`: insert;
   - known `ProductId`: update the changed fields (`PiecesInStock`, `YourMainPrice`,
     `YourPriceLevel1/2`, `QuantityLimitLevel1/2`, `IsActive`, `SinglePackQuantityInPieces`, ...);
   - in your data but not returned: mark as withdrawn (not on offer), do not delete.
7. Store prices as per-100-pieces values together with `CurrencyName`
   ([quantities-and-prices.md](quantities-and-prices.md)).
8. Log the count read against `TotalCount`. If the numbers differ, run the sync again before applying.

Notes:

- Paging is by `ProductId`; if the catalogue changes during the run, a product can be missed or
  seen twice. Staging and the `TotalCount` check cover this.
- If you need inactive products too, use `activeonly = false`.
- Pace the calls; do not run pages in parallel without need.
- Retry a page after an empty `404` or a network error a few times before failing the run.

## 2. Import an order from a CSV file

Goal: customer's CSV with an article number (or EAN) and a quantity becomes an Argip order.

Prerequisite: use an account in the testing phase until the import is proven
(`GET v1/Customer/Info`, `TestingPhase = true`). In the testing phase the writes (`PUT CustomTag`,
`PUT Products`, `PUT ShippingAddress`, `PUT CustomerParts`, `DELETE CustomerParts`) return `Message` and
are not persisted: skip any read-back check, or expect the old state
([customer-data.md](customer-data.md#testing-phase)).

1. Read `GET v1/ShippingAddress/0/1000` once and let the user pick the address (`ShippingAddressId`).
2. For each CSV row, resolve the product to a `ProductId`:
   - by EAN: `GET v1/Products/Ean/{ean}`;
   - by your own index: `GET v1/Products/YourIndex/{index}` (an index containing `/` gives an empty
     `404` and cannot be resolved this way);
   - not resolved (`404` with a `message`): put the row on a "not found" list for the user; do not
     guess.
   - Check `IsActive` of the result. An inactive product will be skipped by the order.
   - Quantities in the CSV must be in pieces. Convert from packages if needed.
   - Drop or report rows with a quantity of 0 or less: the API does not reject them, it orders one
     full package without a warning.
3. Build the list `[{ "ProductId": ..., "QuantityInPieces": ... }]` (at most 500 items per call;
   split larger imports into several orders and tell the user).
4. `POST v1/Order/Calculate` with the list.
5. Show the user the `Items` (with `PiecesOrdered` and `PiecesReceived`), the totals, and **every**
   entry of `WarningMessages`. Let them confirm or correct the file.
6. `POST v1/Order/Save` with `ShippingAddressId`, `CustomerOrderNumber` (unique per import),
   `ExecutionDate` (today or later) and the same items.
7. Check the result: `Items` must not be empty; compare `PiecesReceived` with `PiecesOrdered`;
   show the warnings again (Save can reduce quantities or remove out-of-stock items).
8. Record `CustomerOrderNumber` and the response. If Save failed with a timeout or an empty `404`, do
   not resend automatically ([orders.md](orders.md)).

## 3. Check price and stock by EAN

Goal: answer "what does this barcode cost for us and is it available?".

1. `GET v1/Products/Ean/{ean}` (URL-encode the EAN).
   - `404` with `{"message":"Product with Ean = ... not found"}`: unknown EAN.
   - empty `404`: possible server error, retry this read later.
2. Read from the `ProductModel`:
   - name: `ProductFullName`;
   - availability: `IsActive` and `PiecesInStock` (pieces);
   - price per 100 pieces: `YourMainPrice`, and `YourPriceLevel1/2` from `QuantityLimitLevel1/2`;
   - package size: `SinglePackQuantityInPieces`.
3. Price per piece = price per 100 pieces / 100.
4. For a binding answer for a given quantity, call `POST v1/Order/Calculate` with
   `[{ProductId, QuantityInPieces}]` and read `Items[0].Price`, `PiecesReceived` and any warnings.
   It saves nothing.
5. A multi-pack EAN also resolves, to the same product (single-package EAN is searched first).
