# pingvin-share-x (Arch package)

Arch `PKGBUILD` for [pingvin-share-x](https://github.com/smp46/pingvin-share-x),
a self-hosted file-sharing platform (NestJS backend + Next.js frontend).

## Build & install

```sh
makepkg -si        # build, then install with pacman
```

Needs Node 20 (`nodejs-lts-iron`); skip the test suite with `makepkg --nocheck`.

## Run

```sh
systemctl enable --now pingvin-share-x-backend.service pingvin-share-x-frontend.service
```

- Backend API on `:8080` (prefix `/api`); frontend on `:3000`.
- Put a reverse proxy in front routing `/api/*` → `:8080` and the rest → `:3000`.
- Data: `/var/lib/pingvin-share-x`  ·  Cache: `/var/cache/pingvin-share-x`

## Configure

Via systemd drop-ins only — never edit the shipped units:

```sh
systemctl edit pingvin-share-x-backend.service    # BACKEND_PORT, DATABASE_URL, DATA_DIRECTORY, CLAMAV_*
systemctl edit pingvin-share-x-frontend.service   # PORT, HOSTNAME, API_URL
```

Examples in `/usr/share/doc/pingvin-share-x/`.
