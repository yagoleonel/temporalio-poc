# OIDC Authentication — Temporal UI

The Temporal UI supports OpenID Connect (OIDC) single sign-on via its built-in
auth module. When enabled, users are redirected to an identity provider (IdP) to
authenticate before they can access the UI.

## How it works

1. A user opens the Temporal UI (`http://localhost:8080`).
2. The UI detects that `TEMPORAL_AUTH_ENABLED=true` and redirects the browser to
   the **OIDC provider URL** for authentication.
3. The user authenticates with the IdP (login form, MFA, etc.).
4. The IdP redirects back to the **callback URL** with an authorization code.
5. The UI exchanges the code for tokens (ID token + access token) using the
   **client ID** and **client secret**.
6. The user session is established and the UI is accessible.

## Environment variables

All variables are set in the `.env` file and referenced by `docker-compose.yml`
in the `temporal-ui` service.

| Variable                     | Description                                                                                         |
|------------------------------|-----------------------------------------------------------------------------------------------------|
| `TEMPORAL_AUTH_ENABLED`      | Set to `true` to enable OIDC authentication. When `false` or unset, the UI is open (no login).      |
| `TEMPORAL_AUTH_PROVIDER_URL` | The OIDC discovery endpoint of your IdP (often ends in `/.well-known/openid-configuration` or similar). The UI fetches the authorization, token, and userinfo endpoints from this URL. |
| `TEMPORAL_AUTH_CLIENT_ID`    | The OAuth 2.0 client ID registered with your IdP for the Temporal UI application.                   |
| `TEMPORAL_AUTH_CLIENT_SECRET`| The corresponding client secret. Keep this value in `.env` only — never commit it to version control.|
| `TEMPORAL_AUTH_CALLBACK_URL` | The URL the IdP redirects to after authentication. Must match the redirect URI registered in the IdP. Default for local dev: `http://localhost:8080/auth/sso/callback`. |
| `TEMPORAL_AUTH_SCOPES`       | Space-separated list of OIDC scopes to request. Typical value: `openid profile email`.              |

## Configuration steps

### 1. Register a client in your IdP

Create an OAuth 2.0 / OIDC application in your identity provider with:

- **Application type:** Web application
- **Redirect URI:** `http://localhost:8080/auth/sso/callback`
  (adjust host/port for non-local deployments)
- **Grant type:** Authorization Code
- **Scopes:** `openid`, `profile`, `email`

Note the **client ID** and **client secret**.

### 2. Set the variables in `.env`

```dotenv
TEMPORAL_AUTH_ENABLED=true
TEMPORAL_AUTH_PROVIDER_URL=https://your-idp.example.com/.well-known/openid-configuration
TEMPORAL_AUTH_CLIENT_ID=<client-id>
TEMPORAL_AUTH_CLIENT_SECRET=<client-secret>
TEMPORAL_AUTH_CALLBACK_URL=http://localhost:8080/auth/sso/callback
TEMPORAL_AUTH_SCOPES=openid profile email
```

### 3. Restart the UI

```bash
docker compose up -d temporal-ui
```

Open <http://localhost:8080> — you should be redirected to your IdP login page.

## Disabling authentication

Set `TEMPORAL_AUTH_ENABLED=false` (or remove the variable) in `.env` and restart:

```bash
docker compose up -d temporal-ui
```

## Troubleshooting

| Symptom                              | Check                                                                                    |
|--------------------------------------|------------------------------------------------------------------------------------------|
| Redirect loop after login            | Verify `TEMPORAL_AUTH_CALLBACK_URL` matches the redirect URI registered in the IdP.      |
| 401 / invalid client                 | Confirm `TEMPORAL_AUTH_CLIENT_ID` and `TEMPORAL_AUTH_CLIENT_SECRET` are correct.          |
| "Provider not reachable" on startup  | Ensure `TEMPORAL_AUTH_PROVIDER_URL` is accessible from inside the Docker network.         |
| Scopes error                         | Check that the requested `TEMPORAL_AUTH_SCOPES` are allowed by the IdP client config.    |
