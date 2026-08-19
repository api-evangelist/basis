---
name: Authenticate against the Basis Platform API and make a first read
description: >-
  Obtain an OAuth 2.0 access token from the Basis authorization server and use it
  to make an authenticated read against the Basis Platform API, choosing the right
  grant for whether the caller is a user or a machine.
api: openapi/basis-analytics-api-openapi.yml
operations:
  - GET /v1/me
  - GET /v1/agency
generated: '2026-08-13'
method: generated
source: >-
  openapi/basis-analytics-api-openapi.yml and the Authentication section of the
  Basis Platform API description at https://api.basis.net/swagger.json.
  The specification declares no operationIds, so operations are identified by
  method + path.
---

# Authenticate against the Basis Platform API

## Before you start

Basis credentials are not self-serve. The API is available to Basis (the docs
still say "Centro") customers and integration partners, and an organization must
obtain a client ID and secret from its Basis representative before anything below
will work. Ask for the base URL you have been provisioned for
(`https://api.basis.net/v1` production, `https://api-sandbox.basis.net/v1`
sandbox) and, if you intend to use the authorization-code flow, ask Basis to
register your redirect URI.

OAuth happens on `https://auth.basis.net`, **not** on `https://api.basis.net`.
Getting this wrong is the single most common failure.

## Step 1 — pick the grant

- **Acting on behalf of a signed-in person** → authorization code. This is the
  only grant that returns a refresh token, and the only one under which
  `GET /v1/me` works.
- **Acting as a service, no human present** → client credentials. The application
  must have an owner (agency) ID registered with Basis or the token endpoint
  returns 401. The token covers *every* client in the agency, so if the agency has
  restricted clients, Basis recommends the authorization-code flow instead.
- **Do not use the password grant.** Basis has deprecated it.

## Step 2 — get a token

Client credentials:

```
curl --location 'https://auth.basis.net/oauth/token' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id=<your application ID>' \
  --data-urlencode 'client_secret=<your application secret>' \
  --data-urlencode 'audience=https://api.basis.net'
```

Authorization code — send the user to:

```
https://auth.basis.net/authorize?response_type=code&client_id=<client_id>&scope=openid profile email offline_access&state=<state>&redirect_uri=<redirect URI>&audience=https://api.basis.net
```

then exchange the returned `code` at `https://auth.basis.net/oauth/token` with
`grant_type=authorization_code`.

**Cache the token.** Client-credentials token issuance is capped at **10 per
hour and 25 per day**. Minting a token per request will exhaust the quota within
minutes and return `429`. When you hit that 429, read the Auth0 token-quota
headers on the response before retrying.

## Step 3 — make the call

```
curl -X GET "https://api.basis.net/v1/agency" \
  -H "Authorization: Bearer <access_token>"
```

`GET /v1/agency` is the right smoke test: it works under both grants and returns
the organization the credential belongs to (`id`, `name`, `dsp_advertiser_id`).

`GET /v1/me` is the smoke test for the authorization-code flow specifically. Under
a client-credentials token it returns **404, not 401** — the token represents an
organization rather than a user. Do not treat that 404 as a broken credential.

## Reading the results

Single reads return `{"data": { ... }}`. List reads return
`{"metadata": {"cursor", "page_size", "total"}, "data": [ ... ]}`.

## Failure handling

| Status | Body | What it means | Do this |
|---|---|---|---|
| 401 | `{"message":"missing authorization header"}` | No or bad token | Re-mint; check you sent `Bearer` and the right audience |
| 401 from the **token** endpoint | — | Client-credentials app has no owner/agency ID | Contact Basis support |
| 400 | `{"message":"params/id must match pattern ..."}` | Identifier format wrong for that resource | See the identifier table in `conventions/basis-conventions.yml` |
| 404 with `error`+`statusCode` | `{"message":"Route GET:/v1/nope not found",...}` | The route does not exist | Fix the path |
| 404 with `message` only | — | Resource not visible to this credential | Check the client is not restricted |
| 429 | — | 75,000 req/hour per API user, or the token quota | Back off; reuse tokens; ask Basis to raise the limit |

There is no machine-readable error code — branch on HTTP status, and parse
`message` only as a last resort.

## Related

- `authentication/basis-authentication.yml`
- `scopes/basis-scopes.yml`
- `errors/basis-problem-types.yml`
- `rate-limits/basis-rate-limits.yml`
