# Public search contract

Portable handler: `school_search_api.mjs`. Supabase adapter:
`supabase/functions/school-directory-search/index.ts`.

Supabase route: `GET https://<project-ref>.supabase.co/functions/v1/school-directory-search?q=woodlands`.
A website can mount the handler at a route of its choosing.

Skolo Saka’s existing website adapter at
`https://www.skolosaka.co.za/api/public/schools/search?q=sunnyside` uses its own
`{ schools, total, limit }` response envelope (limit at most 10). The school-search
demo consumes that website adapter. The portable/Supabase adapter below uses
`{ items }` with a maximum of 6; consumers must choose the matching envelope.

```json
{"items":[{"id":"00000000-0000-4000-8000-000000000001","name":"Example Primary School","level":"primary","province":"Gauteng","municipality":"Example municipality","town":"Example town","verified":false}]}
```

This example is fictional. Return only these fields. Consumers should use the
school UUID for selection and show the location to distinguish duplicate names.
Database exports and public API redistribution both need appropriate source-data rights.

- Validate `q`: one value, minimum 2 characters, maximum 100; cap results at 6.
- Use the existing public-directory query/anonymous database permissions.
- Provide bounded timeouts, rate limiting, cache isolation and explicit error responses.
- CORS may allow a demo origin; it is not authentication or abuse prevention.
- On failure return an error with no source SQL or credentials. Never substitute fixtures.
- The importer is not callable by anonymous API clients.

## Errors

The portable handler returns the following statuses. All error responses use
`Cache-Control: no-store` and `Vary: Origin`; when the request origin is in the
allowed list, they also include `Access-Control-Allow-Origin` for that origin.
Bodies below are fictional examples of the exact shape and wording the handler
returns.

| Status | When it happens | JSON body | Important headers |
|---|---|---|---|
| `400` | `q` is missing, duplicated, longer than 100 characters, normalizes to fewer than 2 characters or empty, or contains control characters. Lookup is not attempted. | `{"error":"Provide one school search of 2 to 100 characters."}` | `Vary: Origin`, `Cache-Control: no-store` |
| `403` | An `Origin` header is present but is not in the configured allow-list. Lookup is not attempted and no `Access-Control-Allow-Origin` is sent. | `{"error":"Origin is not allowed."}` | `Vary: Origin`, `Cache-Control: no-store` |
| `405` | The method is neither `GET` nor `OPTIONS`. | `{"error":"Method is not allowed."}` | `Allow: GET, OPTIONS`, `Vary: Origin`, `Cache-Control: no-store` |
| `429` | The per-instance rate limit rejects the request. | `{"error":"Too many searches. Try again shortly."}` | `Retry-After: 60`, `Vary: Origin`, `Cache-Control: no-store` |
| `503` | The search throws, times out, returns a non-array, or returns a row that fails public-field validation. Details are never included in the body. | `{"error":"School search is unavailable. Please try again."}` | `Vary: Origin`, `Cache-Control: no-store` |

An allowed `OPTIONS` preflight returns `204` with an empty body and these headers:
`Access-Control-Allow-Methods: GET, OPTIONS`,
`Access-Control-Allow-Headers: authorization, apikey, content-type, x-client-info`,
`Access-Control-Max-Age: 600`, plus `Vary: Origin`, `Cache-Control: no-store`
and `Access-Control-Allow-Origin` for the allowed origin. An unapproved origin
receives the `403` response above before preflight handling.

A successful `GET` returns `200` with the `{ "items": [...] }` body shown above
and `Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=120`.

Rate limiting, timeout handling and cache state are per runtime instance and are
not shared between instances. No availability guarantee is provided; hosting
operators define their own quotas, availability and data-use terms.

Public dataset access does not grant access to donations, memberships, user contact
details or production administration. Hosting operators define their API quotas,
availability and data-use terms separately from the MIT software licence.
