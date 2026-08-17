---
name: Authenticate against a Bonita runtime and make a first call
description: Establish a Bonita session, capture the CSRF token, and make an authenticated read and write against the Bonita Web REST API without tripping the 401 that catches almost every first integration.
api: openapi/bonitasoft-bonita-openapi.yml
operations: [login, getSession, searchProcesses, logout]
---

# Authenticate against a Bonita runtime

Bonita does not use API keys. Every call rides a **session cookie plus a CSRF
token**, and the CSRF token is where nearly every first integration fails.

## Before you start

Establish the base URL. Bonita is self-hosted, so there is no vendor host.

- Local container: `http://localhost:8080/bonita`
- Bonita Cloud production: `https://{customer-name}.bonitacloud.com/bonita`
- Bonita Cloud non-production: `https://{customer-name}-integration.bonitacloud.com/bonita`
- On premises: `{scheme}://{host}:{port}/bonita`

The API root is `<base>/API`. **Never point this at a host you do not own** — a
Bonita runtime is somebody's live workflow engine.

## Steps

1. **Log in.** `login` — `POST /loginservice`, content type
   `application/x-www-form-urlencoded`, body `username`, `password`,
   `redirect=false`, `redirectURL=`. A success is **204 No Content**, not 200.
   Capture two things from the response:
   - the `JSESSIONID` cookie (`Path=/bonita; HttpOnly; SameSite=Lax`)
   - the `X-Bonita-API-Token` cookie value

2. **Send both on every subsequent request.** The `JSESSIONID` cookie always;
   the `X-Bonita-API-Token` **header** on every `POST`, `PUT` and `DELETE`. Its
   value is the cookie value from step 1. CSRF protection is enabled by default on
   fresh installations, so omitting the header on a write returns 401 even though
   your session is valid.

3. **Confirm the session.** `getSession` — `GET /API/system/session/unusedId`.
   Returns the `Session` object with `user_id` and `session_id`. Use this as the
   liveness check rather than re-logging in.

4. **Make a real read.** `searchProcesses` — `GET /API/bpm/process?p=0&c=10`.
   Note that **`p` and `c` are required**: omit either and you get 400, not a
   defaulted first page. Pagination state comes back in the `content-range`
   response header, and the body is a bare JSON array.

5. **Log out when done.** `logout` — `GET /logoutservice?redirect=false`.

## Rules

- **Always use the newest cookies.** The documentation warns explicitly that the
  cookies transferred must be the ones generated during the *last* successful
  login, and that the `X-Bonita-API-Token` header must match that session's token.
  Re-logging in invalidates the previous pair.
- **401 means re-authenticate; 403 does not.** A 403 means the authenticated
  user's *profile* lacks the permission mapped to that endpoint. Retrying will
  never fix it — an administrator must grant the profile. Check the endpoint
  against the profile permission tables before assuming a bug.
- **Enterprise + OIDC is different.** When the runtime is configured for OIDC SSO,
  the API also accepts `Authorization: Bearer <access_token>` (the `bearer_auth`
  scheme). In that mode refresh the token rather than calling `/loginservice`.
- **Platform administration is a separate session.** `POST /platformloginservice`
  with credentials from `bonita-platform-community-custom.properties`. A tenant
  session cannot reach platform endpoints.
- Errors carry only `{ "message": "..." }`. Branch on the HTTP status, never on
  the message text.

See `authentication/bonitasoft-authentication.yml`,
`conventions/bonitasoft-conventions.yml`, `sandbox/bonitasoft-sandbox.yml`.
