# OAuth2 Authentication (GitHub + Keycloak) via Nginx

Protecting access to a test website using OAuth2/OIDC, with Nginx as a reverse proxy, `oauth2-proxy` as the authentication middleware, and two identity providers: GitHub OAuth and self-hosted Keycloak.

## Architecture

```
Browser → Nginx (:80) → auth_request → oauth2-proxy (Docker) → GitHub / Keycloak
                                              ↓
                                     website (/var/www/...)
```

For public access, a Cloudflare Tunnel was added:

```
External user → Cloudflare → Tunnel → Nginx (WSL) → oauth2-proxy → Keycloak
```

## Repository structure

```
.
├── index.html, index2.html, index4-8.html   — website pages
├── css/, fonts/, js/, images/,
│   owlcarousel/, sources/, unitegallery/     — static assets
├── nginx/default.conf                        — copy of the working Nginx config (IaC)
├── docker-compose.yml                        — oauth2-proxy + Keycloak
└── .gitignore                                 — excludes .env from git
```

Branches:
- `main` — baseline
- `oauth2-github` — working version with GitHub OAuth2
- `oauth2-keycloak` — working version with Keycloak, groups/roles
- `full-site-with-roles` — full multi-page site with access differentiation

## Part 1 — GitHub OAuth2

**Implemented:**
- Static website served by Nginx (`/var/www/aws_oauth2_test_website`)
- Nginx as a reverse proxy with `auth_request` before serving content
- OAuth App registered in GitHub (Client ID/Secret)
- `oauth2-proxy` in Docker, `--provider=github`, access restricted via `--github-user`
- Login/logout tested; unauthorized access confirmed blocked

## Part 2 — Keycloak instead of GitHub

**Implemented:**
- Keycloak (Docker, `quay.io/keycloak/keycloak:26.0`), dedicated Realm
- Confidential Client with a Client Secret
- `oauth2-proxy` switched to `--provider=oidc`, with separate `login-url` / `redeem-url` / `jwks-url` (the issuer seen by the browser and the one used for server-to-server calls differ technically)
- Users, Groups (`site-users`, `site-editors`, `site-admins`), Realm Roles, roles mapped to groups
- Client Scope Mapper (Group Membership) — groups are included in the token

## Part 3 — Group-based access restriction

Implemented via `oauth2-proxy`'s `allowed_groups` parameter (passed as a query param to `/oauth2/auth`), rather than Nginx `if`/`map` (which proved unreliable — `auth_request_set` resolves later than `if` in Nginx's processing order).

| Resource | Allowed groups |
|---|---|
| `/` (homepage + most pages) | any authenticated user |
| `/index5.html` (author page) | `site-editors`, `site-admins` |
| `/nginx-status` (live Nginx stats) | `site-admins` only |

## Part 4 — Public domain access via Cloudflare

- Custom domain connected through Cloudflare, subdomain used for the tunnel
- Cloudflare Tunnel (`cloudflared`) — exposes the local Nginx instance without opening router ports or requiring a static IP
- Nginx got an additional `location /realms/` block to proxy Keycloak through the same public entry point
- Keycloak: `KC_PROXY: edge` — trusts proxy headers instead of requiring its own HTTPS termination
- **Known limitation:** the built-in Keycloak admin console (`/admin/master/console/`) remains local-only (`http://localhost:8080/admin/...`) — its React SPA turned out to be too strict about the HTTPS context relayed through the tunnel. User/Group management is done locally; the console is demonstrated via screen sharing when needed.

## Running it

```bash

docker compose up -d
sudo cp nginx/default.conf /etc/nginx/sites-available/default
sudo nginx -t && sudo service nginx reload
sudo cp -r index*.html css fonts js images owlcarousel sources unitegallery /var/www/aws_oauth2_test_website/
sudo chown -R www-data:www-data /var/www/aws_oauth2_test_website/
```

## Test users (Keycloak realm)

| User | Group | 2FA |
|---|---|---|
| user1 | site-users | yes |
| user2 | site-admins | no |
| user3, user4 | site-editors | no |
| user5, user6, user7 | site-admins | no |

Passwords are kept out of the repository.