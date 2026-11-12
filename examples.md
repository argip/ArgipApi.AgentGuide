# Example code

Working clients for the Argip API, one repository per language. Each one has the same structure:
a token cache with refresh and one retry after 401, typed models named like the schemas in
[`openapi.json`](openapi.json), pagination helpers, a small demo command line tool and a quickstart.

| Language | Repository | Quickstart (run from the repository root) | Demo (run from the repository root) |
|---|---|---|---|
| C# (.NET) | https://github.com/argip/ArgipApi.Examples.CSharp | `dotnet run --project quickstart/ArgipApi.Quickstart` | `dotnet run --project demo/ArgipApi.Demo -- <command>` |
| Python | https://github.com/argip/ArgipApi.Examples.Python | `python quickstart/quickstart.py` | `python -m demo <command>` |
| PHP | https://github.com/argip/ArgipApi.Examples.Php | `php quickstart/quickstart.php` | `php demo/argip.php <command>` |
| TypeScript (Node.js) | https://github.com/argip/ArgipApi.Examples.TypeScript | `npm run quickstart` | `npm run demo -- <command>` |

Install dependencies first (see each repository's README). Set these environment variables, or copy `.env.example` to `.env` in the example repository:

```bash
export ARGIP_CLIENT_ID=...
export ARGIP_CLIENT_SECRET=...
# optional, these are the defaults:
export ARGIP_API_URL=https://argipapi.argip.com.pl/
export ARGIP_TOKEN_URL=https://identityserver.argip.com.pl/connect/token
```

## Warning: the quickstarts write data

A quickstart walks through the whole flow, including `Order/Save`. It **places a real order** unless
your account is in the testing phase (`TestingPhase = true` in `Customer/Info`; see
[docs/orders.md](docs/orders.md)). Run it only with a testing-phase account, or read the code first.
The clients also contain `PUT ShippingAddress`, which creates a real delivery address at Argip. With a
testing-phase account the writes (tags, product mapping, customer parts, addresses) are validated but
not saved, and the response has a `Message` field ([docs/customer-data.md](docs/customer-data.md#testing-phase)).
Every method that writes is commented with what it changes.
