# Authentication

The API uses OAuth 2.0 with the **client credentials** grant (server to server). You get a
`client_id` and a `client_secret` from the Argip sales department (export@argip.com.pl). You trade
them for an access token and send the token with every API call.

## Get a token

`POST https://identityserver.argip.com.pl/connect/token`, body **form-urlencoded**:

```http
POST /connect/token HTTP/1.1
Host: identityserver.argip.com.pl
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET&scope=argipapi
```

Response `200 OK`:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 28800,
  "token_type": "Bearer"
}
```

`expires_in` is in seconds (28800 = 8 hours).

```bash
curl -X POST https://identityserver.argip.com.pl/connect/token \
  -d grant_type=client_credentials -d client_id="$ARGIP_CLIENT_ID" \
  -d client_secret="$ARGIP_CLIENT_SECRET" -d scope=argipapi
```

**The body must be a form, not JSON.** A JSON body is rejected with
`400 {"error":"invalid_request"}`. (Older versions of the API's `/docs` page showed a JSON body; that is wrong.)
In HTTP libraries use the "form" or "data" option (`data=` in Python requests, `URLSearchParams`
in `fetch`, `FormUrlEncodedContent` in .NET, `http_build_query` in PHP).

## Call the API

Send the token in every request:

```http
GET /v1/Customer/Info HTTP/1.1
Host: argipapi.argip.com.pl
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

A missing, malformed or expired token gives `401` with an empty body and a
`WWW-Authenticate` header that starts with `Bearer`.

## Cache and refresh

- **MUST** keep the token in memory and reuse it for all calls (one token serves many requests,
  including parallel ones).
- Remember when it expires (`now + expires_in`) and fetch a new one when less than **5 minutes**
  remain. Do not wait for a `401`.
- If a call returns `401` anyway (for example the token was revoked), fetch a new token **once**,
  retry the call **once**, and if it is still `401` fail with an error: the credentials are wrong or
  disabled.
- With several processes or workers, each can hold its own token; do not share the secret in logs.
- Never request a token per API call: it is slow and wastes the token server's capacity.

## Token server errors

Errors come as `400` with a JSON body, for example `{"error":"invalid_client"}`.

| `error` | Meaning |
|---|---|
| `invalid_client` | Wrong `client_id` or `client_secret`, or the client is disabled |
| `invalid_request` | The request is malformed, for example a JSON body or a missing field |
| `unsupported_grant_type` | `grant_type` is not `client_credentials` |
| `invalid_scope` | `scope` is not `argipapi` |

Do not retry `400` automatically in a loop; fix the request or the credentials.

## Secrets

Keep `client_id` and `client_secret` in environment variables (`ARGIP_CLIENT_ID`,
`ARGIP_CLIENT_SECRET`) or a secret store. Do not commit them, print them or put them in URLs. Add
`.env` to `.gitignore` if you use one.
