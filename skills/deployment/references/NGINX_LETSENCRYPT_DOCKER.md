# NGINX + Let's Encrypt in Docker Compose

The standard "nginx terminates TLS, certbot renews via webroot, all in
Docker Compose" pattern — plus the upstream-image gotchas that bite
when you wrap or override the nginx `command`.

Use this reference when wiring a new project's production stack around
`nginx:*-alpine` (or the regular `nginx:*`) + `certbot/certbot` for
automated Let's Encrypt issuance and renewal.

---

## The pattern

```
public internet  ─►  nginx (TLS, HTTP→HTTPS, security headers, hardening)
                       │
                       └──► proxy_pass to the app container (PHP-FPM /
                            Django / Node / …)

certbot sidecar  ─► writes ACME challenge files to a shared webroot
                    every ~12h; renews when a cert is close to expiry
nginx sidecar    ─► reloads itself every ~6h to pick up renewed certs
```

Three services, three shared bind-mounts:

| Host path        | Mounted into        | Purpose                                   |
| ---------------- | ------------------- | ----------------------------------------- |
| `./certbot/conf` | `/etc/letsencrypt`  | Issued certs + `options-ssl-nginx.conf`   |
| `./certbot/www`  | `/var/www/certbot`  | ACME HTTP-01 webroot                      |
| `./nginx/templates` | `/etc/nginx/templates` | Source-of-truth nginx config templates |

The classic shape (`wmnnd/nginx-certbot` and its forks) wires the
auto-reload + auto-renew with two `while :;` loops as `command:`
overrides. **That override has a sharp edge** — see § "CMD-override
gotcha" below.

---

## CMD-override gotcha — the primary lesson

The official `nginx` Docker image's `/docker-entrypoint.sh` runs the
template-rendering scripts in `/docker-entrypoint.d/*.sh` (including
`20-envsubst-on-templates.sh` which expands `/etc/nginx/templates/*.template`
into `/etc/nginx/conf.d/*.conf`) **only when the container's command
starts with `nginx` or `-`**:

```sh
# excerpt from /docker-entrypoint.sh in the upstream image
if [ "$1" = "nginx" ] || [ "$1" = "nginx-debug" ]; then
    # … run /docker-entrypoint.d/*.sh …
fi
exec "$@"
```

If you override `command:` to wrap nginx in a shell for the auto-reload
loop:

```yaml
# ❌ BROKEN — silently skips template rendering
command:
  - /bin/sh
  - -c
  - |
    while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g 'daemon off;'
```

…`$1` is now `/bin/sh`, the entrypoint scripts don't run, and nginx
starts with the image's stock defaults. **No `app.conf`** in
`/etc/nginx/conf.d/` — only the built-in `default.conf` that serves
`/usr/share/nginx/html`. Symptom: every request 404s with
`server: nginx/<version>` in the response.

For a Let's Encrypt bootstrap this is invisible until the ACME HTTP-01
challenge gets 404'd:

```
Detail: <ip>: Invalid response from
  http://<domain>/.well-known/acme-challenge/<token>: 404
```

It looks like a DNS / firewall / webroot issue. It isn't — the
`/.well-known/acme-challenge/` location was never compiled into the
running config. A real production rate-limit attempt is the price of
discovery if you don't `--staging` first.

### Fix — invoke the entrypoint scripts yourself

```yaml
# ✅ correct
command:
  - /bin/sh
  - -c
  - |
    # nginx upstream entrypoint only auto-runs /docker-entrypoint.d/*.sh
    # when the command starts with 'nginx'; our shell wrapper bypasses
    # that, so we invoke the template-rendering scripts ourselves first.
    for f in /docker-entrypoint.d/*.sh; do [ -x "$$f" ] && "$$f" || true; done
    while :; do sleep 6h & wait $${!}; nginx -s reload; done & exec nginx -g 'daemon off;'
```

The `for` loop runs everything in `/docker-entrypoint.d/` — IPv6 listen,
local resolvers, **envsubst-on-templates**, worker tuning. The `||true`
guard tolerates scripts being non-zero (e.g. on platforms where one
sub-script is a no-op).

Verification: after `docker compose up --force-recreate -d nginx`,
**inside** the container the rendered config must exist:

```bash
docker compose exec nginx cat /etc/nginx/conf.d/app.conf | head -20
```

If that file is missing, the gotcha bit you again — re-check the
`command:` block.

---

## The bootstrap dance

nginx refuses to start if `ssl_certificate` points at a missing file —
so the very first `docker compose up` chicken-and-eggs on a fresh host
with no certs yet. Sequence the bootstrap as five steps in a
`scripts/init-letsencrypt.sh`-style script:

1. **Download recommended TLS params** (idempotent — skip if present):
   ```bash
   curl -sS https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf > certbot/conf/options-ssl-nginx.conf
   curl -sS https://raw.githubusercontent.com/certbot/certbot/master/certbot/certbot/ssl-dhparams.pem      > certbot/conf/ssl-dhparams.pem
   ```
2. **Create a dummy self-signed cert** at the target path so nginx can
   boot on `:443`:
   ```bash
   docker compose run --rm --entrypoint sh certbot -c \
     "mkdir -p /etc/letsencrypt/live/$DOMAIN && \
      openssl req -x509 -nodes -newkey rsa:2048 -days 1 \
        -keyout /etc/letsencrypt/live/$DOMAIN/privkey.pem \
        -out    /etc/letsencrypt/live/$DOMAIN/fullchain.pem \
        -subj   /CN=localhost"
   ```
3. **Start nginx (+ web)** — nginx now boots, serves the ACME location
   over `:80`, and holds the dummy cert in memory for `:443`:
   ```bash
   docker compose up --force-recreate -d web nginx
   ```
4. **Delete the dummy** (forces certbot to issue rather than renew):
   ```bash
   docker compose run --rm --entrypoint sh certbot -c \
     "rm -Rf /etc/letsencrypt/live/$DOMAIN \
             /etc/letsencrypt/archive/$DOMAIN \
             /etc/letsencrypt/renewal/$DOMAIN.conf"
   ```
5. **Request the real cert** (always run `--staging` first to verify the
   pipeline before burning the production rate limit), then reload:
   ```bash
   docker compose run --rm --entrypoint sh certbot -c \
     "certbot certonly --webroot -w /var/www/certbot \
        --email $EMAIL --agree-tos --no-eff-email --staging \
        -d $DOMAIN"
   docker compose exec nginx nginx -s reload
   ```

   Once the staging path issues + nginx serves https://, drop
   `--staging` and re-run for the real, browser-trusted cert.

The dummy step is the bit that's easy to skip and discover at 2 a.m. —
nginx WILL crash-loop on the first `up` without it.

---

## nginx template — the minimum useful shape

```nginx
# ---- HTTP ------------------------------------------------------------
server {
    listen      80;
    listen [::]:80;
    server_name ${DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

# ---- HTTPS -----------------------------------------------------------
server {
    listen      443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name ${DOMAIN};

    ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

    # Mozilla-recommended TLS — let the include be authoritative.
    # Do NOT add manual ssl_protocols / ssl_ciphers / ssl_session_*
    # alongside — they override the curated values with weaker ones and
    # silently break on future certbot updates.
    include     /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options    nosniff always;

    location / {
        proxy_pass http://web:80;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### Filter env vars to `${DOMAIN}` only — and check the variable name

The nginx image's envsubst step expands every env var by default,
which means literal `$host` and `$request_uri` get blown away. Restrict
substitution to your own vars:

```yaml
nginx:
  environment:
    DOMAIN: ${DOMAIN}
    NGINX_ENVSUBST_FILTER: DOMAIN
```

The env var is `NGINX_ENVSUBST_FILTER` — not `NGINX_ENVSUBST_FILTER_VARS`
or `NGINX_ENVSUBST_FILTER_LIST`. The image silently ignores misspellings,
and the breakage only surfaces when an nginx runtime variable happens
to collide with a defined env var. (It "works" by accident otherwise,
which makes the bug latent.)

---

## Hardening the proxy when the repo is bind-mounted

A common compose pattern bind-mounts the project repo into the app
container (`./:/var/www/html`) for live edits. That means **every
infrastructure file in the repo** (`Dockerfile`, `docker-compose*.yml`,
`scripts/`, `nginx/`, `.env*`, `config.*.php`, `README.md`, …) is
sitting next to the app's docroot, and a request like
`https://example.com/scripts/deploy.sh` would proxy through to the app
container and serve it.

Block infrastructure paths at the nginx layer rather than relying on
`.dockerignore` (which doesn't help when the prod compose bind-mounts
the host repo):

```nginx
# Inside the HTTPS server block, BEFORE the catch-all `location /`:

location ~ ^/(scripts|nginx|certbot|docker|docs|\.git|\.superpowers|phpmailer)(/|$) {
    return 404;
}
location ~ ^/(Dockerfile|docker-compose.*\.yml|\.env.*|config(\.[^/]+)?\.php|[^/]+\.md)$ {
    return 404;
}
```

Two regex blocks rather than one keep file-name patterns separate from
directory patterns and make the intent grep-able. The `[^/]+\.md`
pattern catches `README.md`, `DEPLOY.md`, `CHANGELOG.md`, and anything
added later — generalising `README\.md` to "all top-level markdown" was
worth more than a tighter pattern.

### Don't manually duplicate certbot's TLS settings

Once you `include /etc/letsencrypt/options-ssl-nginx.conf`, **don't**
also write your own `ssl_protocols` / `ssl_ciphers` /
`ssl_prefer_server_ciphers` / `ssl_session_*` directives. Manual
values override the curated include — and a typical hand-rolled
`ssl_ciphers HIGH:!aNULL:!MD5` is broader than certbot's curated
forward-secrecy-and-AEAD-only list, silently re-enabling weaker
suites. Pick one source. The include's the right one.

---

## Bind-mount permissions for secret files

When the app container reads a config file (PHP `require config.php`,
Django `python manage.py runserver` reading a `.env`), and the file is
chmod-600 on the host owned by the deploying user (typically UID 1000),
the container's `www-data` (UID 33) **cannot read it
through the bind mount**. Result: PHP fatal "Permission denied" on
every form submission; Django startup error.

Two ways to fix; the inside-container approach is cleanest because
`www-data` is a named group inside the container, GID 33 portable, no
host-side `sudo` needed:

```sh
# In the web container's entrypoint (runs as root inside container):
for f in /var/www/html/config.php /var/www/html/.env.prod; do
  if [ -f "$f" ]; then
    chgrp www-data "$f" 2>/dev/null || true
    chmod 640      "$f" 2>/dev/null || true
  fi
done
```

Owner stays as the deploying user (so they can edit on the host),
group becomes `www-data` (GID 33), mode `640`. The bind mount carries
the chgrp back to the host as numeric GID 33. On most Debian/Ubuntu
hosts that resolves to `www-data` too; on hosts without that group,
it just shows as `33` — harmless.

Don't use host-side `chgrp` — it needs `sudo` (the deploying user is
typically not in group 33 on the host) and isn't portable across
hosts that lack the `www-data` group.

---

## The renewal loop — let it auto-renew

After bootstrap, the steady-state services are:

```yaml
nginx:
  command:
    - /bin/sh
    - -c
    - |
      for f in /docker-entrypoint.d/*.sh; do [ -x "$$f" ] && "$$f" || true; done
      while :; do sleep 6h & wait $${!}; nginx -s reload; done & exec nginx -g 'daemon off;'

certbot:
  entrypoint:
    - /bin/sh
    - -c
    - |
      trap exit TERM
      while :; do certbot renew --quiet; sleep 12h & wait $${!}; done
```

- `certbot renew` is a no-op until a cert is within ~30 days of
  expiry. Running every 12 h is cheap and gives a 60-attempt window
  before expiry.
- nginx reload every 6 h picks up the freshly-renewed cert. (nginx
  doesn't watch the cert file — without the reload, an expired cert
  stays loaded in memory.)
- Both loops `wait $${!}` on the sleep so a `docker stop` signal
  terminates promptly rather than waiting out the sleep.

No host cron required.

---

## Operational hand-back checklist

When deploying this stack to a fresh server, the bootstrap script
should verify in order:

1. DNS A record points at the server's public IP (`dig +short $DOMAIN`).
2. Ports `80` and `443` open to the public internet (cloud firewall +
   `ufw`).
3. `config.php` / `.env.prod` exist and are filled before
   `init-letsencrypt.sh` runs — refuse to proceed if missing.
4. First cert request runs with `--staging`. Only switch to
   production after staging issues successfully (real LE allows 5
   cert failures per hostname per hour; staging is loose).
5. After the real cert installs, the user smoke-tests the actual
   workload (form submit, login flow, whatever the app does) — a
   green TLS handshake doesn't prove the app works.

---

## Anti-patterns

- ❌ Overriding nginx `command:` without invoking
  `/docker-entrypoint.d/*.sh` — templates silently don't render,
  ACME challenges 404.
- ❌ Starting with `--staging=0` for the first deploy on a new host
  — burns the production rate limit on every misconfiguration. Real
  hosts cost 1 hour per 5 failures.
- ❌ Hardcoding `ssl_protocols` / `ssl_ciphers` alongside
  `include options-ssl-nginx.conf` — your manual values override the
  curated ones, silently re-enabling weaker suites.
- ❌ `chmod 600` host-side on bind-mounted secret files without
  fixing group perms inside the container — every request to the app
  fails with EACCES.
- ❌ Trusting `.dockerignore` to protect infrastructure files when the
  prod compose uses a bind mount — `.dockerignore` is ignored by bind
  mounts. Use nginx `location` blocks.

---

**Last Updated**: 2026-05-25 — first ship from a live production
debug session. Originating context: a Docker-Compose + Let's Encrypt
project's PR #3 (CMD-override fix in `docker-compose.prod.yml`) after
a real Let's Encrypt cert request failed the ACME challenge with the
symptom that gave this gotcha its name.
