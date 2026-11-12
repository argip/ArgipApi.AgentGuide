# Quantities and prices

## The two rules

1. **Quantities are in pieces.** `PiecesInStock`, `QuantityInPieces`, `SinglePackQuantityInPieces`,
   `QuantityLimitLevel1/2`, `PiecesOrdered` and `PiecesReceived` all count pieces.
2. **Prices are per 100 pieces.** `MainPrice`, `YourMainPrice`, `YourPriceLevel1`, `YourPriceLevel2`
   and `OrderModelItem.Price` are the price of 100 pieces, in the currency named in
   `CurrencyName`.

So the net value of a line is `Price * PiecesReceived / 100`, rounded to 2 decimals (half away
from zero). `ItemNetValue` already contains it.

Illustrative numbers: a screw with `YourMainPrice = 12.50` costs 12.50 per 100 pieces, 0.125 per
piece. 400 pieces cost `12.50 * 400 / 100 = 50.00`.

## Which price applies

Each product has up to three of your prices:

| Quantity (pieces) | Price per 100 pieces |
|---|---|
| below `QuantityLimitLevel1` (or no level 1) | `YourMainPrice` |
| from `QuantityLimitLevel1` | `YourPriceLevel1` |
| from `QuantityLimitLevel2` | `YourPriceLevel2` |

A level exists only when its limit and its price are both above 0. The order calculation picks the
level from the quantity actually received (`PiecesReceived`, after rounding to full packages).
`MainPrice` is Argip's list price before your discount, shown for reference.

Prices are already in your currency and include both `Discount` and `ExtraDiscount` from
`Customer/Info`; do not apply either again.

## Full packages

Products are sold in whole packages of `SinglePackQuantityInPieces` pieces:

- A quantity up to one package becomes exactly one package, **without a warning**. This also holds
  for `0` and negative quantities: they are not rejected but order one full package. Never send a
  quantity of 0 to mean "skip".
- A larger quantity is rounded **up** to whole packages. If the rounded quantity is more than you
  asked for and still within stock, the response has a warning of type 3.
- Items with the same `ProductId` are added up before rounding (`PiecesOrdered` shows the sum).

Example: package of 200 pieces. You send 250. You get 400 (two packages) and the warning
`Revised quantity from 250 to 400. Only full packages can be sold!`.

## Stock

Orders are checked against `PiecesInStock` of the product. See [orders.md](orders.md) for what
happens when you ask for more than the stock (differs between `Calculate` and `Save`).

## VAT and totals

- `TaxRate` is a percentage (`23` means 23 %). In an order item it is 0 when your account is
  charged no VAT.
- `TotalNetValue` is the sum of `ItemNetValue`.
- `TotalGrossValue` is computed per VAT rate: the net values of all items with the same rate are
  added up and the VAT is added to this sum, rounded to 2 decimals (half to even, unlike
  `ItemNetValue`); the results are summed. It can
  differ by a cent from adding the gross value of every item.
- `TaxValue = TotalGrossValue - TotalNetValue`.
- Weights: `ItemWeight` per item, `TotalWeight` is their sum. Use these for the shipping weight; do
  not derive it from `BoxWeight`, which is a different value.

## Pitfalls

- Do not treat `Price` as a unit price. Divide by 100 for a per-piece price.
- Do not send quantities in packages or boxes; the API wants pieces.
- Use `decimal` types for money (not binary floating point) and keep the currency with the amount.
- A price of 0 on a level means that level does not exist.
