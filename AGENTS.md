# Agent guide: Argip Client API

## What this API is

The Argip Client API is a REST API for business customers of Argip. It lets you browse the product
catalogue, read stock levels and your own prices, calculate orders and place them. It speaks JSON
over HTTPS and is protected with OAuth 2.0 (client credentials).

- API base URL: `https://argipapi.argip.com.pl/` (routes start with `v1/`)
- Interactive reference (Swagger UI): `https://argipapi.argip.com.pl/swagger/ui/index.html`
- Machine-readable schema: [`openapi.json`](openapi.json) in this repository. Where its required/nullable
  flags differ from these docs, the docs win ([docs/endpoints.md](docs/endpoints.md))
- Token endpoint: `https://identityserver.argip.com.pl/connect/token`
- Credentials (`client_id`, `client_secret`): contact the Argip sales department (export@argip.com.pl).

## Mandatory rules for any integration

1. **MUST** cache the access token and reuse it until 5 minutes before it expires (it lives 8 hours).
   Never request a token per API call. After a `401`, refresh the token once and retry the call once.
2. **MUST NOT** hard-code or log secrets (`client_secret`, tokens). Read them from environment
   variables or a secret store.
3. **MUST** send quantities in **pieces** and read prices as **per 100 pieces**
   ([docs/quantities-and-prices.md](docs/quantities-and-prices.md)).
4. **MUST** call `POST v1/Order/Calculate` before `POST v1/Order/Save`, and show its
   `WarningMessages` to the user.
5. **MUST** inspect `Items` and `WarningMessages` of every `Order/Save` response. `200 OK` does
   not mean the order was saved: when every item is rejected, nothing is saved and the status is
   still `200`.
6. **MUST** develop and test order placing with an account in the testing phase
   (`TestingPhase = true` in `GET v1/Customer/Info`). Without it `Order/Save` creates a real order.
   In the testing phase the other writes (custom tags, product mapping, customer parts, shipping
   addresses) are validated but not saved, and their response has a `Message` field
   ([docs/customer-data.md](docs/customer-data.md#testing-phase)).
7. **MUST** page lists with `pagesize` between 1 and 1000 and follow `X-Pagination` (`TotalPages`).
8. **MUST** treat an empty `404` as a possible server error (the API also returns it for unhandled
   failures): log it and, for reads, retry later; do not conclude that the data does not exist. It is
   also the answer for an empty list. **MUST NOT** automatically resend `Order/Save` or any other
   write after an empty `404` or a timeout: the outcome is unknown and a retry can duplicate the order.
9. **MUST NOT** send more than 500 items in one order or calculation.
10. On a production account `PUT v1/ShippingAddress` creates a real delivery address in Argip systems: do not call it just to test. In the testing phase the data is validated but not saved ([docs/customer-data.md](docs/customer-data.md#testing-phase)).
11. Use the lowercase `v1` in every URL. `V1` returns an empty `404`.
12. Send write requests with `Content-Type: application/json`; without it the API returns an empty `415`.
13. `CustomerOrderNumber` is at most 30 characters; longer makes Save fail with an empty `404`.
14. A quantity of 0 or less is not rejected: it orders one full package. Never send it.
15. `ExecutionDate` must be today or later **in Europe/Warsaw**. Compute "today" in that time zone
    and send `yyyy-MM-ddT00:00:00` without an offset.
16. Never send `null` in a write body: send `""` for an unused string. `NewShippingAddress.Email` and
    `Order.Items` are required; a `null` gives an empty `404`.
17. Respect field lengths in writes: `PUT ShippingAddress` and `PUT CustomerParts` silently cut longer
    strings, `PUT Products` and `PUT CustomTag` reject them with `400`
    ([customer-data.md](docs/customer-data.md), [products.md](docs/products.md)).
18. `NewShippingAddress.ReceivingCompanyName` is required (not empty); send `""` only for strings that
    may be empty (`FiendlyName`, `City`).

## Documentation map

| File | What it covers |
|---|---|
| [docs/authentication.md](docs/authentication.md) | Getting a token, caching it, refreshing, token server errors |
| [docs/endpoints.md](docs/endpoints.md) | Every route and every model field |
| [docs/products.md](docs/products.md) | Finding products, paging, EAN and your own index, categories, custom tags |
| [docs/quantities-and-prices.md](docs/quantities-and-prices.md) | Pieces, packages, price levels, discounts, VAT, totals |
| [docs/orders.md](docs/orders.md) | Calculate and Save, warning types 1-5, test mode, limits |
| [docs/customer-data.md](docs/customer-data.md) | Account info, shipping addresses, customer parts |
| [docs/errors.md](docs/errors.md) | Status codes and bodies, the empty 404, retry advice |
| [docs/recipes.md](docs/recipes.md) | Catalogue sync, CSV order import, price and stock by EAN |
| [skills/argip-api/SKILL.md](skills/argip-api/SKILL.md) | Skill with a checklist, for Claude Code and compatible agents |

## Example code

Clients in C#, Python, PHP and TypeScript: see [examples.md](examples.md).

## Keeping this guide current

Source of truth: the Argip API (ArgipApi) and token server code maintained by Argip. Do not
document behaviour that the API does not have. When this guide and `https://argipapi.argip.com.pl`
disagree, the live API wins; report the difference to Argip.
