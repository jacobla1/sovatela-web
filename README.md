# sovatela-web

Static download site for the **Sovatela** desktop app, served at
<https://sovatela.anaubi.com>. Deployed to `/srv/apps/sovatela` on the Pi.

The nginx server block for the subdomain lives in the separate **`web`** repo
(kept nginx-only) — this repo is just the site content.

## Contents

- `index.html` — the download landing page (per-OS buttons + SHA-256 section)
- `assets/` — images/icons for the page
- `downloads/` — installer binaries at runtime (**not** tracked in git)

## Deploy (on the Pi)

```sh
# first time — clone straight into the nginx docroot:
git clone git@github.com:jacobla1/sovatela-web.git /srv/apps/sovatela

# later — update the page:
cd /srv/apps/sovatela && git pull
```

The `downloads/` binaries are gitignored, so they persist across `git pull`s
and never dirty the working tree.

### TLS / nginx (one-time, in the `web` repo)

The `sovatela.anaubi.com` block ships with `:443` commented out. After the
docroot exists:

```sh
sudo nginx -t && sudo systemctl reload nginx        # :80 only — ACME reachable
sudo certbot certonly --webroot -w /var/www/html -d sovatela.anaubi.com
# uncomment the :443 sovatela block in web/nginx/nginx.conf, then:
sudo nginx -t && sudo systemctl reload nginx
```

## Publishing a release

Installers come from the app's GitHub Actions release (attached to the GitHub
Release) — they are **not** committed here.

1. Download the release assets (`.dmg` / `.msi` / `.AppImage` / `.deb`).
2. Copy them into `/srv/apps/sovatela/downloads/`.
3. Generate checksums: `cd downloads && sha256sum * > SHA256SUMS.txt`
4. If filenames changed, update the `href`s and the SHA-256 cells in `index.html`.

nginx `autoindex` lists whatever is in `downloads/`.
