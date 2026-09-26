# Altered Re:Union — Keycloak theme & client examples

This repo holds the **custom Keycloak theme** used by `auth.altered.re` and an
**example frontend** (keycloak-js SPA) that demonstrates how to integrate a
third-party app with the Altered Re:Union identity realm.

```
build/themes/altered/       # the custom theme (login + email + CSS + messages)
frontend-example/           # keycloak-js demo app, switchable between local & prod
dev/                        # local dev seed (realm-export.json + cleanup script)
docker-compose.dev.yml      # one-shot local stack: Keycloak (H2, dev mode) + nginx
```

---

## 1. Working on the theme locally

The dev stack runs a Keycloak in `start-dev` mode with an embedded H2 database,
mounts `build/themes` as a read-only volume, and imports a pre-seeded realm with
two verified test users. Theme files are hot-read — edit a `.ftl`, `.css` or
`.properties` and just refresh the browser, no rebuild needed.

### Starting the stack

```sh
docker compose -f docker-compose.dev.yml up
```

| Service       | URL                                      | Credentials            |
|---------------|------------------------------------------|------------------------|
| Keycloak      | http://localhost:8080                    | `admin` / `admin` (master realm) |
| Admin console | http://localhost:8080/admin              | same |
| Example app   | http://localhost:8000/**?backend=local** | — |

> The example app defaults to the **prod** backend (see §3). When working on
> templates locally, always open it with `?backend=local` so it authenticates
> against your local Keycloak — otherwise the login buttons will redirect you
> to `auth.altered.re`.

Two pre-verified test users live in the `players` realm:

| Username | Password            | Email                |
|----------|---------------------|----------------------|
| `alice`  | `TestPassword1234`  | alice@example.test   |
| `bob`    | `TestPassword1234`  | bob@example.test     |

*(Passwords must satisfy the realm policy: 14 chars, upper + lower + digits.)*

### Where to look when editing a template

Every login screen Keycloak renders corresponds to one `.ftl` template. The
custom ones live in [build/themes/altered/login/](build/themes/altered/login/).
If a screen is unstyled, it means we haven't overridden that template yet
and Keycloak falls back to the near-empty `base` theme.

Reference material:

- **Stock templates to copy-from**:
  https://github.com/keycloak/keycloak/tree/26.0.6/themes/src/main/resources/theme/base/login
- **Stock English messages** (to know which message keys exist):
  https://github.com/keycloak/keycloak/blob/26.0.6/themes/src/main/resources/theme/base/login/messages/messages_en.properties
- **Server-side FTL context** (which variables are available in each template):
  https://www.keycloak.org/docs/latest/server_development/#_themes
- **Theme overview docs**:
  https://www.keycloak.org/docs/latest/server_development/#_themes

Typical override workflow:
1. Find the stock template on GitHub, copy it into `build/themes/altered/login/`.
2. Strip the `<@layout.registrationLayout>` macro wrapper — our theme writes
   full `<html>` documents instead, to keep CSS self-contained.
3. Keep the form action URL, hidden inputs and message keys from the stock
   template untouched — they're what Keycloak's server-side code expects.
4. Refresh the browser, verify the screen renders, check that form submission
   still lands on the next step of the flow.

---

## 2. Re-syncing the realm export from prod

The file `dev/realm-export.json` is **derived from the production realm**.
When the prod config changes (new client, new scope, new flow, password
policy tweak…), re-sync it so collaborators get an accurate local setup.

### Steps

1. **Export** from the prod admin console:

   Realm settings → Action (top-right) → **Partial export**
   → tick *Include clients* → **Export**.

   This downloads a JSON file. **Do not commit it as-is** — it contains the
   prod SMTP config, all built-in Keycloak clients, and internal UUIDs.

2. **Drop the raw file** into `dev/` as `realm-export-raw.json`
   (that's the filename the cleanup script expects).

3. **Run the cleanup**:

   ```sh
   node dev/clean.js
   ```

   This produces a sanitized `dev/realm-export.json` by:
   - stripping every `id` / `containerId` (Keycloak regenerates them on import)
   - removing `smtpServer` (no mail server in dev, avoids leaking the prod provider)
   - removing the 6 Keycloak built-in clients (`account`, `account-console`,
     `admin-cli`, `broker`, `realm-management`, `security-admin-console`) —
     Keycloak recreates them automatically at import
   - removing built-in auth flows, authenticator configs, and scope mappings
   - forcing `verifyEmail: false` so registration works without SMTP
   - injecting the two test users (`alice` / `bob`)

4. **Delete the raw file** once you've verified the cleaned output
   (`git status` should only show `dev/realm-export.json` as modified).

5. **Smoke-test** by restarting the dev stack with a fresh volume:

   ```sh
   docker compose -f docker-compose.dev.yml down -v
   docker compose -f docker-compose.dev.yml up
   ```

6. Commit the updated `dev/realm-export.json`.

---

## 3. Connecting a new app to the prod realm

The example frontend in [frontend-example/](frontend-example/) is set up to
point at `https://auth.altered.re` **by default**. That's the quickest way to
sanity-check that your real client id and scopes are set up correctly before
you wire them into your actual app.

### Running the example against prod

1. Serve the example directory on port 8000 (any static server works):

   ```sh
   docker compose -f docker-compose.dev.yml up frontend
   # or, without Docker:
   cd frontend-example && python -m http.server 8000
   ```

2. Open http://localhost:8000 — no query param needed, the SPA already
   targets `https://auth.altered.re`.

3. Click **Login** / **Register** — you'll hit the real prod realm.

   (To temporarily switch back to the local Keycloak, add `?backend=local`
   to the URL.)

### Testing your own client id and scopes

Edit [frontend-example/app.js](frontend-example/app.js) and change the `prod`
entry of `KEYCLOAK_CONFIG`:

```js
const KEYCLOAK_CONFIG = {
  local: { url: "http://localhost:8080",   realm: "players", clientId: "localhost-test" },
  prod:  { url: "https://auth.altered.re", realm: "players", clientId: "YOUR-CLIENT-ID" },
};
```

Then update the `scope` option passed to `keycloak.login()` further down in
the same file:

```js
document.getElementById("loginBtn").onclick = () =>
  keycloak.login({ scope: "email read-collection" });
```

Currently available optional scopes on the `players` realm:

| Scope              | Purpose                                       |
|--------------------|-----------------------------------------------|
| `email`            | access to the user's email claim              |
| `read-decks`       | read access to the user's decks               |
| `read-collection`  | read access to the user's card collection     |
| `write-deck`       | write access to decks *(not for example)*     |
| `backup-collection`| offline backup access *(requires approval)*   |

The `write-deck` scopes is not available on the localhost-test client for security reasons — contact Altered Re:Union to enable them on your client.

The `backup-collection` scope is not doing anything in itself. It's just a way to know the user agrred to share the backup of his collection with you app. We will send backups of your collection to these apps to not lose data in case the Re:Union server should go extinct for some reason. This feature does not exists yet.

### Registering a new client id

New client ids are provisioned manually in the prod Keycloak admin console.
Ask an Altered Re:Union admin with these informations:
- a `clientId` (public client, PKCE required)
- Valid Redirect URIs and Web Origins for your app
- the optional scopes your app needs

---

## 4. Token claims: `pseudo` in the access token

The player's public name is the `pseudo` user attribute (required at signup,
unique). The realm uses the email as the Keycloak username
(`registrationEmailAsUsername: true`), so `preferred_username` is **not** a
public name: never display it.

The `pseudo` protocol mapper lives on the **`profile` client scope**
(`dev/realm-export.json` → `clientScopes[name=profile].protocolMappers[name=pseudo]`).
It is added to the ID token, userinfo, introspection **and the access token**
(`"access.token.claim": "true"`). Resource servers that only read the Bearer
access token, like the Decks API
([altered-core-decks-api#63](https://github.com/Altered-Community/altered-core-decks-api/pull/63)),
need the claim there to show deck authors.

A client only gets `pseudo` if the `profile` scope is in its token: assigned as
a default client scope, or assigned as optional and requested (`scope=openid profile`).

### Applying it in prod

Use the admin console (one field, applied instantly, reversible). Do **not**
re-import `dev/realm-export.json` or run a partial import with "Overwrite":
the export is older than the prod realm and would reset later changes.

1. Admin console → realm **players** → **Client scopes** → **profile** → **Mappers** tab.
2. Open **pseudo**.
3. Switch **Add to access token** to **On**. Leave every other toggle as is
   (ID token On, userinfo On, introspection On, lightweight access token Off).
4. **Save**.

If a client has its own `pseudo` mapper (Clients → *client* → Client scopes →
*client*-dedicated → Mappers), apply the same toggle there.

### Side effects

- Tokens already issued keep their claims. New access tokens get `pseudo` on
  the next login or refresh (access token lifespan: 5 minutes).
- Token size grows by the length of one short string claim.
- Every client whose token includes the `profile` scope, and every API that
  receives those access tokens, now sees `pseudo`. These clients could already
  read it from the ID token and userinfo.
- No change to email exposure: the `email`, `preferred_username` and other
  mappers are untouched.

### Verifying

Print claim **names** only, never values:

```sh
# ACCESS_TOKEN = a fresh access token (log out and back in first)
python3 -c 'import sys,json,base64; p=sys.argv[1].split(".")[1]; d=json.loads(base64.urlsafe_b64decode(p+"="*(-len(p)%4))); print(sorted(d)); print("pseudo present:", "pseudo" in d)' "$ACCESS_TOKEN"
```

Expected: `pseudo present: True`.

Then make one authenticated call to the Decks API with that token (for
example `GET /api/decks?itemsPerPage=1`) so it stores the pseudo, and check the
public list:

```sh
curl -s 'https://decks.alteredcore.org/api/decks/public?itemsPerPage=100' \
  | python3 -c 'import sys,json; m=json.load(sys.stdin)["member"]; print("with username:", sum(1 for d in m if isinstance(d["user"],dict) and d["user"].get("username")), "/", len(m))'
```

Expected: `with username` grows as authors log in again (or all at once after
`app:users:sync-pseudos` on the Decks API).
