# Secure Frontend App Example

## Local Setup

Open this repository in its VS Code dev container. The course image is
`ghcr.io/tpdavison/devops-labs:2026-latest`, with Node.js 22 and npm. For a
host-based setup, use Node.js 22 as declared in `package.json`.

Install the versions recorded in the lockfile:

```bash
npm ci
```

### Compile and Hot-Reload for Development

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). Vite listens on all
interfaces for dev-container access and uses port 5173 without selecting a
different port if it is busy.

### Type-Check, Compile and Minify for Production

```bash
npm run build
```

This runs TypeScript checking and creates `dist/`. To preview that build:

```bash
npm run preview -- --host 0.0.0.0
```

`npm run type-check` checks types separately; `npm run format` formats source
files with the project's Prettier preferences. Production hosting must serve
`index.html` for client-side routes because this SPA uses history routing.

## Shared Auth0 and backend setup

Complete the [security backend's ordered Auth0 setup](https://github.com/tpdavison/cis3039-example-security-svc/blob/main/readme.md#auth0-setup)
through steps 1–5 first. It is the single setup route for both repositories:
reuse the tenant, Products API, reader role, two test users and SPA registration.
Do not create another tenant or a second set of users here. Choose your own test
passwords in the Auth0 Dashboard; never put passwords or access tokens in Git,
commands saved in files, or screenshots.

Before switching to this repository, confirm:

- The Products API uses RS256, defines `read:products`, and has **Enable RBAC**
  switched on. Leave **Add Permissions in the Access Token** off.
- Only the reader test user has a role granting `read:products`; the second user
  has no role or direct grant of that permission.
- The backend uses the same API identifier/audience and exact tenant issuer
  (including the issuer's final slash), and has been rebuilt/restarted or republished.
- One **Single Page Application** is registered in that tenant with the test
  users' database connection enabled. Its **Allowed Callback URLs**,
  **Allowed Logout URLs** and **Allowed Web Origins** include
  `http://localhost:5173`. Add the actual deployed SPA HTTPS origin to all three
  lists when deploying; retain localhost if still using it.
- The backend separately permits the SPA origin in its CORS list. Auth0's origin
  settings do not configure backend CORS. CORS and HTTPS do not authorise users.

The flow is **role → API permissions → requested/granted scope → backend check**.
The SPA requests `openid profile email read:products`; the backend checks granted
`scope`, without requiring an optional `permissions` claim. Signing in alone
therefore does not grant access to prices. See the official
[RBAC guidance](https://auth0.com/docs/get-started/apis/enable-role-based-access-control-for-apis)
and [Vue SDK setup](https://auth0.github.io/auth0-vue/).

The SDK uses Authorization Code with PKCE and its default in-memory session
behaviour. This example does not configure persistent `localStorage` or refresh
tokens. If session restoration or token acquisition fails, the app shows an
error and asks you to sign in again; it never silently retries as anonymous.

### Configure and run the SPA

In a Bash terminal in **this SPA repository's course Dev Container**, install
with `npm ci` as above, then copy the public example settings:

```bash
cp .env.example .env.local
```

Edit `.env.local` using the values from the shared setup:

```dotenv
VITE_AUTH0_DOMAIN=YOUR_ACTUAL_AUTH0_DOMAIN
VITE_AUTH0_CLIENT_ID=YOUR_SPA_CLIENT_ID
VITE_AUTH0_AUDIENCE=YOUR_PRODUCTS_API_IDENTIFIER
VITE_API_BASE_URL=https://YOUR_ACTUAL_TEST_HOSTNAME/api
```

The audience is the Auth0 API identifier, not the backend HTTP address. A SPA
needs no client secret or Function App host key. All `VITE_*` values are public
and embedded in the browser build. Follow Week 5 for assessment configuration
practice rather than adding secrets here.

Use the backend deployment's recorded `SECURITY_API_BASE_URL`. If you need to
recover the hostname, sign in to Azure in the intended subscription and run:

```bash
read -r -p 'Security Function App name: ' SECURITY_APP_NAME
read -r -p 'Security resource group: ' SECURITY_RESOURCE_GROUP
SECURITY_HOSTNAME=$(az functionapp show \
  --name "$SECURITY_APP_NAME" \
  --resource-group "$SECURITY_RESOURCE_GROUP" \
  --query defaultHostName --output tsv)
printf '%s\n' "$SECURITY_HOSTNAME"
```

Stop if the command fails or returns an empty hostname. Set the base URL to
`https://` followed by that actual hostname and `/api`; do not construct the
hostname from the resource name. For a running local backend, use
`http://localhost:7071/api` instead.

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173). Restart Vite after changing
settings. For a deployed SPA, rebuild with `npm run build` before uploading the
contents of `dist/` using the Week 5 hosting workflow.

### Verify access and authentication transitions

Use the browser Network panel to inspect `/api/products`, without capturing
credentials or bearer tokens in screenshots. The same backend route serves both
public and protected reads; the backend enforces price access.

| Request | Backend result | SPA result |
| --- | --- | --- |
| Signed out; no bearer token | `200`, `pricePence: null` in JSON | Public products with price placeholders. |
| Invalid or expired bearer token | `401`, no Product data | Authentication error; no public fallback. |
| Signed in as the user without `read:products` | `403`, no Product data | Permission-denied message. |
| Signed in as the reader with granted `read:products` | `200`, including prices | Products with prices. |

For direct invalid/expired-token and response-JSON checks, follow step 6 of the
backend's shared setup. A verification-service failure is an operational error,
not invalid credentials; it must not become an anonymous success.

Also check the following in the browser:

1. On initial SDK loading, product requests wait for the session check.
2. Sign in with each user, then sign out. Sign-out clears the product view before
   redirecting; the returned signed-out page fetches public products.
3. Throttle requests in the Network panel and change auth state while a request
   is pending. An earlier response must not restore prices or replace newer data.
4. After changing role/permission grants, sign out and sign back in to obtain a
   newly issued token. Retry uses the SDK's normal token caching behaviour.
5. Check session/token failures and a stopped/unavailable backend: errors must
   be visible, with no stale products or anonymous fallback. Retry after fixing
   API availability; for session failures, sign out and sign in again. For `403`,
   check the role, API identifier, requested scope and user grants.

These are rehearsal checks, not an automated-testing lab. A successful build
alone does not prove deployed Azure/Auth0 behaviour. Keep the shared backend,
receiver and Auth0 test setup until all checks are complete; then remove only
disposable identities/resources no longer needed by other work.

The example demonstrates public catalogue data and permission-dependent prices.
It does not implement assessment staff-only actions, ownership rules or
reservation safeguards. Map these examples into the existing Stage 2 plan and
diagrams; this repository adds no assessment deliverable or Week 7 coding task.

## How the project was made

```bash
npm create vue@3 . -- \
  --force \
  --bare \
  --typescript \
  --router \
  --prettier
```

Restore any deleted devcontainer configuration:

```bash
git restore .devcontainer/devcontainer.json
```

Add extensions to devcontainer:

- `"esbenp.prettier-vscode"`,
- `"Vue.volar"`

Update preferences in `.prettierrc.json`:

- `"semi": true`
- `"singleQuote": true`
- `"printWidth": 80`

## Baseline product listing

The baseline SPA reads products from an existing HTTP service; it does not add product
editing, storage, automatic retries or a fake-data mode.

Copy `.env.example` to `.env.local` and set the products API base path:

```env
VITE_API_BASE_URL=http://localhost:7071/api
```

`/products` is appended to this path. A final slash is optional: both `/api` and
`/api/` select `/api/products`. If the setting is absent, the intentional default
is the local Functions host shown above. An explicitly blank or invalid value
fails startup; only absolute HTTP(S) URLs without credentials, query strings or
fragments are accepted. Use `/api` for the companion Functions service.

Restart Vite after changing environment files. Vite embeds these **public,
build-time** settings in `dist/`; deployment changes require a rebuild. Never put
server credentials or function keys in browser settings. Local environment files
are ignored; `.env.example` contains only public placeholders.

The existing service contract is a bare JSON array, for example:

```json
[
  { "id": "p1", "name": "Notebook", "pricePence": 399 },
  { "id": "p2", "name": "Pen", "pricePence": null }
]
```

IDs must be unique, non-blank strings; names must be non-blank strings. Prices
may be absent or null, or a non-negative safe integer in pence. Descriptions are
optional strings. Additional service fields are ignored. Missing prices display
as a placeholder; the browser does not calculate which prices a caller may see.
An invalid list or record is an operational failure, never an empty-list success.
Network and HTTP failures show a safe message and a manual Retry button. Each
refresh clears previous data; failures display an error without stale prices.
The display preserves service order and makes no sorting or pagination promise.

| File | Responsibility |
| --- | --- |
| `src/domain/product.ts` | Read-only product list projection, without Vue or wire concerns. |
| `src/app/product-service.ts`, `list-products.ts` | Service port and ordinary use-case function with explicit dependencies. |
| `src/infra/http-product-service.ts` | HTTP request, status handling and validation of unknown JSON. |
| `src/config/composition-root.ts` | Resolve public environment settings and assemble the product service and Auth0 plugin. |
| `src/composables/products-key.ts`, `use-products.ts` | Typed injection, use-case calls, loading and safe errors. |
| `src/main.ts`, `src/views/Products.vue` | Bootstrap dependencies/startup feedback and render UI state. |

The Auth0 integration acquires an access token only for signed-in requests and
passes it through the service port. Session failures stop the request. The HTTP adapter attaches the bearer
header; it does not acquire tokens or make authorisation decisions.
