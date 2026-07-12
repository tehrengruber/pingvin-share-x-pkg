# Maintainer: Till Ehrengruber <till@ehrengruber.ch>
pkgname=pingvin-share-x
pkgver=1.19.0
pkgrel=3
pkgdesc="Self-hosted file sharing platform — Pingvin Share X fork (NestJS backend + Next.js frontend)"
arch=('x86_64')  # bundles native modules (argon2, sharp) and prisma query engine
url="https://github.com/smp46/pingvin-share-x"
license=('BSD-2-Clause')
# Pinned to Node 24 LTS, matching upstream's production runtime (Dockerfile
# `node:24-alpine`). 1.19.0 requires node >=22 (nestjs-i18n); CI runs the system
# tests on node:22, so node 24 is a superset. Not the current `nodejs` (26) —
# untested upstream.
depends=('nodejs-lts-krypton')
makedepends=('npm' 'git' 'python')  # python + base-devel for argon2/node-gyp
install="$pkgname.install"
source=("$pkgname-$pkgver.tar.gz::https://github.com/smp46/pingvin-share-x/archive/refs/tags/v$pkgver.tar.gz"
        "$pkgname-backend.service"
        "$pkgname-frontend.service"
        "$pkgname.sysusers"
        "$pkgname.tmpfiles"
        "0001-email-localize-fromnow.patch")
# Run `updpkgsums` after editing the local files to replace the SKIPs.
sha256sums=('13f4f46303bf7cf83e8156efa42eb99ec28addad8ce8aeb286c76f3e24af1719'
            '1d4c227daf30ebb7897bfe692baed83db22c0a66b7fe947e10847db9d0fd916f'
            '4bf0124bbe9cb19c10fa1774430b6d8ada40bbca644039422eb2d708caac4961'
            '3b7fed3c716a02a81743d68484d70087fe19647717ea1285889901c995e7a1df'
            '0262c657493fb61cac9685583d85a0bd65787b3fecf1a8e8b258a6e8a91b6694'
            'a05abc48fcfc055cba3b5627709ff13c243e2580f5a61705b5136e68a37d00c5')

prepare() {
  cd "$srcdir/$pkgname-$pkgver"

  # Localize email expiry time (moment().fromNow()) and i18n fallbacks: upstream
  # hardcodes `const lang = ""` in email.service.ts, overriding the configured
  # general.defaultLanguage so emails always render in English. Fixed upstream's
  # own stated intent in PR #70. Drop when merged upstream.
  patch -Np1 -i "$srcdir/0001-email-localize-fromnow.patch"
}

build() {
  cd "$srcdir/$pkgname-$pkgver"

  export npm_config_cache="$srcdir/npm-cache"
  export NEXT_TELEMETRY_DISABLED=1

  # Backend (NestJS).
  cd backend
  npm ci
  npx prisma generate
  npm run build
  # Precompile the prisma seed to plain JS so it runs under `node` at runtime
  # instead of ts-node. The seed is one self-contained file (imports only
  # @prisma/client + crypto). This lets us drop dev deps from the shipped
  # package below. tsc may exit non-zero on type-check noise but still emits;
  # the file check enforces that emission actually happened.
  npx tsc prisma/seed/config.seed.ts \
    --rootDir prisma/seed --outDir prisma/seed \
    --module commonjs --target ES2021 --esModuleInterop --skipLibCheck || true
  [ -f prisma/seed/config.seed.js ]
  # Drop devDependencies from what we ship: they're only needed to build
  # (nest/tsc/webpack/eslint/…), and the seed now runs as compiled JS — no
  # ts-node at runtime. Cuts package size and most of the audited vulns /
  # install-script surface. prisma + @prisma/client are production deps, so the
  # CLI, query engine, and generated client survive the prune.
  npm prune --omit=dev
  cd ..

  # Frontend (Next.js).
  cd frontend
  npm ci
  npm run build
}

check() {
  cd "$srcdir/$pkgname-$pkgver/backend"

  # Upstream's only automated test is the newman system suite, run by
  # backend/package.json "test:system" (.github/workflows/backend-system-tests.yml).
  # We mirror that script by hand so we can test the built artifact
  # (`node dist/src/main`) instead of the dev server, isolate the DB and port,
  # and always stop the server afterwards — all in a throwaway sqlite DB. Keep
  # in sync if upstream changes "test:system". Skip with `makepkg --nocheck`.
  local port=18080
  local testdir="$srcdir/check"
  rm -rf "$testdir"; install -d "$testdir/data"

  export NODE_ENV=production
  export DATA_DIRECTORY="$testdir/data"
  export DATABASE_URL="file:$testdir/pingvin-share.db?connection_limit=1"
  export BACKEND_PORT="$port"
  # Writable npm cache for `npx newman` below (mirrors build(); the default
  # ~/.npm may be unwritable in a clean build environment).
  export npm_config_cache="$srcdir/npm-cache"
  # node_modules/.bin on PATH so the bare `prisma` call resolves. Dev deps were
  # pruned in build(), so this stage uses only production deps (+ npx/curl) —
  # i.e. it validates exactly what gets packaged.
  export PATH="$PWD/node_modules/.bin:$PATH"

  # Schema + default config, exactly as the backend unit's ExecStartPre does
  # (prisma is a production dep; the seed runs as precompiled JS under node).
  prisma migrate deploy
  node prisma/seed/config.seed.js

  node dist/src/main &
  local server_pid=$!
  local rc=0
  # wait-on and newman were dev deps (pruned above); run both via npx, as
  # upstream's test:system does. Needs network, like npm ci in build().
  npx --yes wait-on -t 60000 "http://localhost:$port/api/configs" \
    && npx --yes newman run ./test/newman-system-tests.json \
         --env-var "API_URL=http://localhost:$port/api" \
    || rc=$?
  kill "$server_pid" 2>/dev/null || true
  wait "$server_pid" 2>/dev/null || true
  return $rc
}

package() {
  cd "$srcdir/$pkgname-$pkgver"

  local appdir="$pkgdir/usr/lib/$pkgname"
  install -d "$appdir"

  # Backend (NestJS): ship the production tree. build() already pruned dev deps
  # and precompiled the seed (prisma/seed/config.seed.js), so this is the
  # runtime-only set: dist/, production node_modules, prisma schema + migrations.
  cp -a backend "$appdir/"

  # Frontend (Next.js): ship the `output: "standalone"` bundle and run it with
  # `node server.js`, exactly as upstream's Dockerfile + entrypoint.sh do. The
  # standalone tree is self-contained with its own trimmed node_modules (~65M
  # vs ~560M for the full install); Next does NOT copy static assets or public/
  # into it, so they're placed alongside server.js by hand (mirrors the
  # Dockerfile's COPY of .next/static and public into the standalone root).
  install -d "$appdir/frontend"
  cp -a frontend/.next/standalone/. "$appdir/frontend/"
  install -d "$appdir/frontend/.next/static"
  cp -a frontend/.next/static/. "$appdir/frontend/.next/static/"
  cp -a frontend/public "$appdir/frontend/"

  # /usr/lib stays read-only; redirect the app's in-tree writable paths to /var
  # (matches the docker-compose bind mounts).
  #  - uploaded/branding images (docker mounts ./data/images -> public/img)
  mv "$appdir/frontend/public/img" "$appdir/frontend/public/img.default"
  ln -s "/var/lib/$pkgname/data/images" "$appdir/frontend/public/img"
  #  - Next.js runtime cache (image optimization / ISR)
  rm -rf "$appdir/frontend/.next/cache"
  ln -s "/var/cache/$pkgname" "$appdir/frontend/.next/cache"

  # systemd integration
  install -Dm644 "$srcdir/$pkgname-backend.service"  "$pkgdir/usr/lib/systemd/system/$pkgname-backend.service"
  install -Dm644 "$srcdir/$pkgname-frontend.service" "$pkgdir/usr/lib/systemd/system/$pkgname-frontend.service"
  install -Dm644 "$srcdir/$pkgname.sysusers"         "$pkgdir/usr/lib/sysusers.d/$pkgname.conf"
  install -Dm644 "$srcdir/$pkgname.tmpfiles"         "$pkgdir/usr/lib/tmpfiles.d/$pkgname.conf"

  # Example drop-in overrides (configuration is done via `systemctl edit`,
  # never by editing the shipped units).
  install -d "$pkgdir/usr/share/doc/$pkgname"
  cat > "$pkgdir/usr/share/doc/$pkgname/backend-override.conf.example" <<'EOF'
# Drop-in override for the backend.
#   systemctl edit pingvin-share-x-backend.service
[Service]
#Environment=BACKEND_PORT=8080
#Environment=DATA_DIRECTORY=/var/lib/pingvin-share-x/data
#Environment=DATABASE_URL=file:/var/lib/pingvin-share-x/data/pingvin-share.db?connection_limit=1
#Environment=CLAMAV_HOST=127.0.0.1
#Environment=CLAMAV_PORT=3310
EOF
  cat > "$pkgdir/usr/share/doc/$pkgname/frontend-override.conf.example" <<'EOF'
# Drop-in override for the frontend.
#   systemctl edit pingvin-share-x-frontend.service
[Service]
#Environment=PORT=3000
#Environment=HOSTNAME=0.0.0.0
#Environment=API_URL=http://localhost:8080
EOF

  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
