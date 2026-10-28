# Multi-stage Dockerfile for Umbrel Bitcoin Node app
#
# * dev: `docker compose up` builds the **dev** stage only
# hot-reloading backend container (the UI dev server lives in its own
# apps/ui/Dockerfile; both share a hoisted node_modules volume in the compose file).
#
# * production: `docker build ...` produces a single self-contained image with backend + UI + bitcoind
# single final image that contains
#  – compiled backend
#  – bundled React UI (Vite output → dist/public)
#  – compiled shared libs (libs/settings, libs/shared-types)
#  – bitcoind + bitcoin-cli binaries

###########################################
# Bitcoind
###########################################
# The one bitcoind this image ships: the published Bitcoin Knots SHA256 release
# (https://github.com/paulscode/knots-sha256/releases), Bitcoin Knots' patch series
# maintained on Bitcoin Core for the SHA256 chain, consensus-identical to Bitcoin
# Core. The binaries are the reproducible Guix build, downloaded and checked here
# against the maintainer's pinned key (FF76 D484 3EBD 7FA0 6D92 DC0C B8AB 7B8E
# 7E28 0E1A): a SHA256SUMS that does not verify, or a tarball that does not match
# it, fails the build. Same check the StartOS package's Dockerfile makes.
#
# Upstream ships twelve Knots builds and a picker. Every one of them is a
# different chain or an older series than this; the picker is kept with one entry.
FROM debian:stable-slim AS bitcoind_knots_sha256
ARG KNOTS_SHA256_VERSION=31.1.knots-sha256.20261018
ARG TARGETARCH
COPY apps/backend/knots-sha256-maintainer.asc /tmp/maintainer.asc
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates wget gpg gpg-agent \
 && case "${TARGETARCH}" in \
      amd64) host=x86_64-linux-gnu ;; \
      arm64) host=aarch64-linux-gnu ;; \
      *) echo "unsupported TARGETARCH ${TARGETARCH}" >&2; exit 1 ;; \
    esac \
 && cd /tmp \
 && url="https://github.com/paulscode/knots-sha256/releases/download/v${KNOTS_SHA256_VERSION}" \
 && tarball="bitcoin-${KNOTS_SHA256_VERSION}-${host}.tar.gz" \
 && wget -q "${url}/${tarball}" "${url}/SHA256SUMS" "${url}/SHA256SUMS.asc" \
 && export GNUPGHOME=/tmp/gnupg && mkdir -m 700 "$GNUPGHOME" \
 && gpg --batch --quiet --import /tmp/maintainer.asc \
 && gpg --batch --status-fd 1 --verify SHA256SUMS.asc SHA256SUMS > gpg.status \
 && grep -q '^\[GNUPG:\] VALIDSIG FF76D4843EBD7FA06D92DC0CB8AB7B8E7E280E1A ' gpg.status \
 && grep -q '^\[GNUPG:\] GOODSIG ' gpg.status \
 && grep " ${tarball}\$" SHA256SUMS | sha256sum -c - \
 && mkdir -p /out && tar -xzf "${tarball}" -C /out --strip-components=1 \
      "bitcoin-${KNOTS_SHA256_VERSION}/bin/bitcoind" "bitcoin-${KNOTS_SHA256_VERSION}/bin/bitcoin-cli" \
 && /out/bin/bitcoind -version | head -1

##########################################################
# Dependencies layer — install every workspace deps
# Functions as a shared cache for dev & app-builder stages
##########################################################
FROM node:24-slim@sha256:55fb5f5a527ea2c5c4a95f963e48ef90aa38c7b346054fdc8ee9957a2b55d205 AS deps
WORKDIR /repo

# Copy only dependency manifests to keep this layer cache-friendly
COPY package*.json ./
COPY apps/backend/package*.json apps/backend/
COPY apps/ui/package*.json apps/ui/

RUN npm ci --workspaces --include-workspace-root --install-strategy=hoisted

# Add npm binaries to PATH so subsequent stages can invoke tsc / vite / etc.
ENV PATH="/repo/node_modules/.bin:$PATH"

####################################################
# DEV-ONLY build stage: Backend hot-reload image
# - Not referenced by later COPY --from=… steps,
#   so it’s skipped during a plain `docker build`.
# - `docker compose up` targets it via `target: dev`
#   in the docker-compose.yml file.
####################################################
FROM deps AS dev
WORKDIR /repo/apps/backend

VOLUME /data

# bitcoind binaries (the directory name is the version string the settings store)
RUN mkdir -p /opt/bitcoind/v31.1.knots-sha256.20261018
COPY --from=bitcoind_knots_sha256 /out/bin/bitcoind    /opt/bitcoind/v31.1.knots-sha256.20261018/bitcoind
COPY --from=bitcoind_knots_sha256 /out/bin/bitcoin-cli /opt/bitcoind/v31.1.knots-sha256.20261018/bitcoin-cli

# Give 1000:1000 write permisions to /opt/bitcoind (just directory) so we can flip the symbolic link while the container runs as 1000:1000
RUN chown 1000:1000 /opt/bitcoind

# Symbolic link for default bitcoind binary
RUN ln -sfn /opt/bitcoind/v31.1.knots-sha256.20261018 /opt/bitcoind/current
RUN ln -sfn /opt/bitcoind/current/bitcoind    /usr/local/bin/bitcoind
RUN ln -sfn /opt/bitcoind/current/bitcoin-cli /usr/local/bin/bitcoin-cli

# Symlink for datadir so bitcoin-cli auto-finds the RPC cookie
# Allows exec'ing bitcoin-cli inside the container without specifying credentials
RUN ln -sfn /data/bitcoin /root/.bitcoin
RUN mkdir -p /home/node \
 && ln -sfn /data/bitcoin /home/node/.bitcoin \
 && chown -h 1000:1000 /home/node/.bitcoin

CMD ["npm","run","dev"]

#################################################
# app-builder stage
#################################################
FROM deps AS app-builder
WORKDIR /repo
COPY . .

# build shared libraries (libs/*/dist)
# build backend (ts → js) (apps/backend/dist)
# bundle UI (vite build) (apps/ui/dist)
RUN tsc -b libs/shared-types libs/settings apps/backend \
 && npm run --workspace apps/ui build

########################################
# runtime stage — final production image
########################################
FROM node:24-slim@sha256:55fb5f5a527ea2c5c4a95f963e48ef90aa38c7b346054fdc8ee9957a2b55d205 AS runtime
WORKDIR /app

# Install production dependencies only
# Note: import-map lives in package.json, so is also needed so Node can resolve "#settings/#types"
  COPY package*.json ./
  COPY apps/backend/package*.json apps/backend/
  RUN npm ci --omit=dev --install-strategy=hoisted --ignore-scripts

# Backend JS bundle
COPY --from=app-builder /repo/apps/backend/dist ./dist

# Pre-bundled React UI → backend serves this from /public/*
COPY --from=app-builder /repo/apps/ui/dist ./dist/public

# Compiled shared libs (referenced at runtime via import-map)
COPY --from=app-builder /repo/libs/settings/dist ./libs/settings/dist
COPY --from=app-builder /repo/libs/shared-types/dist ./libs/shared-types/dist

# bitcoind binaries (the directory name is the version string the settings store)
RUN mkdir -p /opt/bitcoind/v31.1.knots-sha256.20261018
COPY --from=bitcoind_knots_sha256 /out/bin/bitcoind    /opt/bitcoind/v31.1.knots-sha256.20261018/bitcoind
COPY --from=bitcoind_knots_sha256 /out/bin/bitcoin-cli /opt/bitcoind/v31.1.knots-sha256.20261018/bitcoin-cli

# Give 1000:1000 write permisions to /opt/bitcoind (just directory) so we can flip the symbolic link while the container runs as 1000:1000
RUN chown 1000:1000 /opt/bitcoind

# Symbolic link for default bitcoind binary
RUN ln -sfn /opt/bitcoind/v31.1.knots-sha256.20261018 /opt/bitcoind/current
RUN ln -sfn /opt/bitcoind/current/bitcoind    /usr/local/bin/bitcoind
RUN ln -sfn /opt/bitcoind/current/bitcoin-cli /usr/local/bin/bitcoin-cli

# Symlink for datadir so bitcoin-cli auto-finds the RPC cookie
# Allows exec'ing bitcoin-cli inside the container without specifying credentials
RUN ln -sfn /data/bitcoin /root/.bitcoin
RUN mkdir -p /home/node \
 && ln -sfn /data/bitcoin /home/node/.bitcoin \
 && chown -h 1000:1000 /home/node/.bitcoin

 RUN apt-get update && apt-get install -y \
    curl


ENV NODE_ENV=production

# Data dirs inside /data
ENV BITCOIN_DIR=/data/bitcoin
ENV APP_STATE_DIR=/data/app

# 3000 = HTTP API, 8332 = RPC, 8333 = P2P
EXPOSE 3000 8332 8333
# app and bitcoind data
VOLUME /data

CMD ["node", "dist/server.js"]