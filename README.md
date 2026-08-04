# sovatela-web

Static download site for the **Sovatela** desktop app, served at
<https://sovatela.anaubi.com>. Deployed to `/srv/apps/sovatela` on the Pi.

The nginx server block for the subdomain lives in the separate **`web`** repo
(kept nginx-only) — this repo is just the site content.

## Do not edit the HTML here

**`index.html` and the policy pages are generated.** Their source is
`deploy/web/` in the private **`Scale`** repo, and they are produced by
`node deploy/web/build.mjs <artifacts-dir>` there.

This repo is the *publish* target: it holds the build output so that what is
live corresponds to a commit. To change the page, edit
`Scale/deploy/web/index.html` or the markdown under `Scale/docs/`, rebuild, and
copy the result here.

An edit made directly in this repo will be silently overwritten by the next
release. Worse, the checksums in `index.html` are computed from the installer
bytes — hand-editing them recreates the exact failure `build.mjs` was written
to prevent: a verification step that looks authoritative and proves nothing.

## Contents

- `index.html` — the download landing page (per-OS buttons + SHA-256 section)
- `accessibility/index.html` — generated from `Scale/docs/ACCESSIBILITY.md`
- `assets/` — images/icons. **Hand-maintained, not generated**
- `downloads/` — installer binaries at runtime (**not** tracked), plus
  `SHA256SUMS.txt` (generated, tracked)

`/privacy`, `/terms` and `/security` are deliberately not published. The
`PAGES` list in `Scale/deploy/web/build.mjs` records why for each.

## Publishing a release

Installers come from the app's GitHub Actions release. They are **not**
committed here — `.gitignore` keeps them out while tracking `SHA256SUMS.txt`.

```sh
# 1. In the Scale repo: build against the artifacts as published, not a local build.
gh release download v1.1.1 --repo jacobla1/Scale --dir /tmp/sovatela-release
node deploy/web/build.mjs /tmp/sovatela-release

# 2. Copy the output here and commit — this is the record of what goes live.
cp    deploy/web/dist/index.html      ../sovatela-web/
cp -R deploy/web/dist/accessibility   ../sovatela-web/
cp    deploy/web/dist/SHA256SUMS.txt  ../sovatela-web/downloads/
cd ../sovatela-web && git add -A && git commit && git push

# 3. Installers to the Pi — these cannot come through git.
scp /tmp/sovatela-release/Sovatela_1.1.1_* /tmp/sovatela-release/*.rpm rpi:~/
ssh rpi 'sudo -u appuser mv ~/Sovatela_* /srv/apps/sovatela/downloads/'

# 4. Pages to the Pi — interactively; see "Working on the Pi" below for why.
ssh rpi
sudo -iu appuser
cd /srv/apps/sovatela && git pull

# 5. Verify on the Pi, against the bytes actually being served.
cd downloads && sha256sum -c SHA256SUMS.txt
```

Keep steps 3 and 4 together. `SHA256SUMS.txt` arrives via the pull and the
installers via scp, so doing one without the other leaves the page advertising
hashes for files that are not there.

Do not copy `Sovatela_universal.app.tar.gz` — it is Tauri's updater bundle,
there is no updater, and it only confuses anyone browsing `/downloads/`.

## Verify from outside the LAN

The site sits behind NAT that does not hairpin, so machines on the LAN reach it
through `/etc/hosts` entries pointing at the local address. **A local check
proves nothing about what the public sees, in either direction.** Check from
mobile data with wifi off: valid certificate, every download button returns a
file, `SHA256SUMS.txt` matches the page.

## Working on the Pi

Log in as the owner and work interactively:

```sh
ssh rpi
sudo -iu appuser
cd /srv/apps/sovatela
```

Two reasons this is not a one-liner:

**Ownership.** The tree belongs to `appuser`, so git run as `jbl` refuses with
*"detected dubious ownership"*. Add a `safe.directory` exception only if you
want `jbl` writing files nginx's owner cannot manage.

**The key has a passphrase.** `appuser`'s GitHub key must be unlocked at the
prompt, so a non-TTY `ssh rpi 'sudo -u appuser git … pull'` cannot work — ssh
offers the key, cannot sign with it, and GitHub reports *"Permission denied
(publickey)"*. That message reads like an unregistered key and will send you
looking for a missing deploy key that was never needed. Deploys here are
deliberately hands-on; there is no unattended pull.

First-time clone:

```sh
git clone git@github.com:jacobla1/sovatela-web.git /srv/apps/sovatela
```

The `downloads/` binaries are gitignored, so they persist across pulls and
never dirty the working tree.

### TLS / nginx (one-time, in the `web` repo)

The `sovatela.anaubi.com` block ships with `:443` commented out. After the
docroot exists:

```sh
sudo nginx -t && sudo systemctl reload nginx        # :80 only — ACME reachable
sudo certbot certonly --webroot -w /var/www/html -d sovatela.anaubi.com
# uncomment the :443 sovatela block in web/nginx/nginx.conf, then:
sudo nginx -t && sudo systemctl reload nginx
```

nginx `autoindex` lists whatever is in `downloads/`.
