# Plane-oauth

This document describes a local modification to the Plane codebase that adds a
**generic, fully-configurable OAuth 2.0 (authorization-code) authentication
provider**, alongside the existing provider-specific integrations (Google,
GitHub, GitLab, Gitea).

Instead of hard-coding a single identity provider, the feature lets an instance
operator plug in **any OAuth 2.0 / OpenID Connect provider** (Keycloak, Auth0,
Okta, Azure AD, GitHub, GitLab, ...) by configuring the authorize/token/userinfo
endpoints and the JSON field mapping of the userinfo response.

---

## What was added

### Backend (`apps/api/plane`)

| Area | Files |
| --- | --- |
| Provider | `authentication/provider/oauth/oauth.py` — `GenericOAuthProvider` (subclass of `OauthAdapter`) |
| App views | `authentication/views/app/oauth.py` — `GenericOauthInitiateEndpoint`, `GenericOauthCallbackEndpoint` |
| Space views | `authentication/views/space/oauth.py` — `GenericOauthInitiateSpaceEndpoint`, `GenericOauthCallbackSpaceEndpoint` |
| Routes | `authentication/urls.py`, `authentication/views/__init__.py` |
| Adapter | `authentication/adapter/oauth.py` (error mapping), `authentication/adapter/base.py` (`ENABLE_OAUTH_SYNC`), `authentication/adapter/error.py` (`OAUTH_PROVIDER_ERROR = 5129`) |
| Instance config | `utils/instance_config_variables/core.py` — 16 new `OAUTH_*` config keys |
| Instance API | `license/api/views/instance.py` — exposes `is_oauth_enabled` and `oauth_provider_name` |
| Models | `db/models/user.py` (`Account.provider`), `db/models/social_connection.py` (`medium`) — `("oauth", "OAuth")` |
| Migration | `db/migrations/0123_alter_account_provider_and_more.py` |
| Tests | `tests/unit/auth/test_generic_oauth.py` |

### Frontend

| Area | Files |
| --- | --- |
| Shared types | `packages/types/src/instance/auth.ts` (`oauth` mode, `IS_OAUTH_ENABLED`, `TInstanceOauthAuthenticationConfigurationKeys`), `instance/base.ts` (`is_oauth_enabled`, `oauth_provider_name`) |
| Shared constants | `packages/constants/src/auth/core.ts` (label), `auth/index.ts` (`OAUTH_PROVIDER_ERROR = "5129"`) |
| Shared utils | `packages/utils/src/auth.ts` — error message + banner list |
| Web sign-in | `apps/web/core/hooks/oauth/core.tsx`, `apps/web/helpers/authentication.helper.tsx` |
| Space sign-in | `apps/space/hooks/oauth/core.tsx`, `apps/space/helpers/authentication.helper.tsx` |
| Admin (God Mode) | `apps/admin/hooks/oauth/core.tsx`, `hooks/oauth/index.ts`, `components/authentication/oauth-config.tsx`, `app/(all)/(dashboard)/authentication/oauth/form.tsx` + `page.tsx`, `app/routes.ts`, `components/common/header/core.ts` |

---

## Authentication flow

1. User clicks the **"Sign in with {OAuth Provider Name}"** button on the web or
   space sign-in screen.
2. The browser is redirected to `GET /auth/oauth/?next_page=...`
   (`/auth/spaces/oauth/` for space) which builds the provider authorize URL with
   `client_id`, `redirect_uri`, `response_type=code`, `scope`, and a signed
   `state` token, then redirects to the configured `OAUTH_AUTH_URL`.
3. After the user consents, the provider redirects back to
   `GET /auth/oauth/callback/` with a `code`.
4. The backend exchanges the `code` for tokens at `OAUTH_TOKEN_URL`
   (`grant_type=authorization_code`), then fetches the user info from
   `OAUTH_USERINFO_URL` using the access token.
5. The configured JSON field paths extract `id`, `email`, name, and avatar from
   the userinfo response; the email is optionally validated against
   `OAUTH_EMAIL_VERIFIED_FIELD`.
6. The account is created / logged in via the shared `OauthAdapter` and the user
   is redirected to the original `next_page` (validated to prevent open
   redirects).

---

## Configuration

### Environment variables (`.env`)

These are read as defaults and seeded into the `AppConfig` table on the next
`configure_instance` run; they can be overridden at runtime from the admin UI.

| Variable | Default | Description |
| --- | --- | --- |
| `IS_OAUTH_ENABLED` | `0` | Master switch for the sign-in button |
| `OAUTH_PROVIDER_NAME` | `OAuth` | Display name shown on the sign-in button |
| `OAUTH_CLIENT_ID` | — | OAuth client ID (required) |
| `OAUTH_CLIENT_SECRET` | — | OAuth client secret (required, stored encrypted) |
| `OAUTH_AUTH_URL` | — | Authorize endpoint URL (required) |
| `OAUTH_TOKEN_URL` | — | Token endpoint URL (required) |
| `OAUTH_USERINFO_URL` | — | Userinfo endpoint URL (required) |
| `OAUTH_SCOPE` | `openid email profile` | Space-separated scopes |
| `OAUTH_ID_FIELD` | `sub` | JSON path to the user id in the userinfo response |
| `OAUTH_EMAIL_FIELD` | `email` | JSON path to the email |
| `OAUTH_EMAIL_VERIFIED_FIELD` | *(empty)* | JSON path to the email-verified flag |
| `OAUTH_FIRST_NAME_FIELD` | `given_name` | JSON path to the first name |
| `OAUTH_LAST_NAME_FIELD` | `family_name` | JSON path to the last name |
| `OAUTH_AVATAR_FIELD` | `picture` | JSON path to the avatar URL |
| `OAUTH_ENABLE_EMAIL_VERIFICATION` | `0` | Set to `1` to reject unverified emails (requires `OAUTH_EMAIL_VERIFIED_FIELD`) |
| `ENABLE_OAUTH_SYNC` | `0` | Whether user attributes are re-synced on every sign-in |

Defaults are OIDC-style (`sub`, `email`, `given_name`, `family_name`,
`picture`), so they work out of the box with any standards-compliant OIDC
provider (e.g. Keycloak, Auth0, Google).

### Field mapping

Field values are resolved from the userinfo JSON using **dotted paths**, e.g.
`OAUTH_EMAIL_FIELD=user.email` reads `payload["user"]["email"]`. This supports
providers that nest the user object.

### Admin UI (God Mode)

Navigate to **Settings → Authentication → OAuth**:

- Enable/disable the provider (`IS_OAUTH_ENABLED`).
- Configure client id/secret, the three endpoints, scope, provider name, and all
  field mappings.
- Toggle **"Require a verified email before sign in"**
  (`OAUTH_ENABLE_EMAIL_VERIFICATION`).
- Copy the exact **callback URL** (`{APP_URL}/auth/oauth/callback/`) that must be
  registered in the identity provider.

---

## Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/auth/oauth/` | Initiate web OAuth (redirect to provider) |
| `GET` | `/auth/oauth/callback/` | OAuth callback (web) |
| `GET` | `/auth/spaces/oauth/` | Initiate space OAuth |
| `GET` | `/auth/spaces/oauth/callback/` | OAuth callback (space) |

Callback URL to register with the provider:
`https://{your-plane-domain}/auth/oauth/callback/`

---

## Error codes

| Code | Constant | Meaning |
| --- | --- | --- |
| `5104` | `OAUTH_NOT_CONFIGURED` | Missing `OAUTH_CLIENT_ID`/`SECRET`/`AUTH_URL`/`TOKEN_URL`/`USERINFO_URL` |
| `5124` | `OAUTH_PROVIDER_UNVERIFIED_EMAIL` | Email not verified and verification is enforced |
| `5129` | `OAUTH_PROVIDER_ERROR` | Generic provider error (e.g. missing id field) |

---

## Verification

- `ruff check` and `ruff format` pass on all modified backend files.
- `python -m py_compile` passes (syntax OK).
- Unit tests: `apps/api/plane/tests/unit/auth/test_generic_oauth.py` cover config
  loading, auth URL building, token exchange, userinfo parsing, and
  verified-email enforcement. Run them via the Docker test stack
  (`docker compose -f docker-compose-test.yml ...`).
- The patch for this whole feature is captured in `generic-oauth.patch`
  (apply with `git apply generic-oauth.patch`).