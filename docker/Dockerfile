# ============================
# Stage 1: Build dependencies
# ============================
FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3 AS builder-deps

WORKDIR /app

# Install system dependencies
RUN apk --no-cache --update add \
  alpine-sdk gmp-dev automake libtool inotify-tools autoconf \
  python3 file gcompat libstdc++ curl ca-certificates git make bash \
  nodejs npm

# Copy mix files
COPY mix.exs mix.lock ./
COPY apps/block_scout_web/mix.exs ./apps/block_scout_web/
COPY apps/explorer/mix.exs ./apps/explorer/
COPY apps/ethereum_jsonrpc/mix.exs ./apps/ethereum_jsonrpc/
COPY apps/indexer/mix.exs ./apps/indexer/
COPY apps/utils/mix.exs ./apps/utils/
COPY apps/nft_media_handler/mix.exs ./apps/nft_media_handler/

ENV MIX_ENV=prod
ENV MIX_HOME=/opt/mix

# Fetch dependencies
RUN mix local.hex --force && \
    mix do deps.get, local.rebar --force, deps.compile --skip-umbrella-children

# Copy full source
COPY config ./config
COPY rel ./rel
COPY apps ./apps

# Build frontend assets
RUN cd apps/block_scout_web/assets && \
    npm install && npm run deploy && \
    cd ../../../ && mix phx.digest

# ============================
# Stage 2: Build release
# ============================
FROM builder-deps AS builder

RUN mix compile && \
    mkdir -p /opt/release && \
    mix release blockscout && \
    mv _build/prod/rel/blockscout /opt/release

# ============================
# Stage 3: Final container
# ============================
FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3

WORKDIR /app

# Add non-root user
ARG BLOCKSCOUT_USER=blockscout
ARG BLOCKSCOUT_GROUP=blockscout
ARG BLOCKSCOUT_UID=10001
ARG BLOCKSCOUT_GID=10001

RUN apk --no-cache --update add jq curl && \
  addgroup --system --gid ${BLOCKSCOUT_GID} ${BLOCKSCOUT_GROUP} && \
  adduser --system --uid ${BLOCKSCOUT_UID} --ingroup ${BLOCKSCOUT_GROUP} --disabled-password ${BLOCKSCOUT_USER}

# Set release version
ARG RELEASE_VERSION=8.1.1
ENV RELEASE_VERSION=${RELEASE_VERSION}

# Copy built release and configs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /opt/release/blockscout .
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/config_helper.exs ./config/config_helper.exs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/config_helper.exs /app/releases/${RELEASE_VERSION}/config_helper.exs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/runtime.exs /app/releases/${RELEASE_VERSION}/runtime.exs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/assets/precompiles-arbitrum.json ./config/assets/precompiles-arbitrum.json

# Create runtime folders
RUN mkdir dets temp && chown -R ${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app

USER ${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP}

# Cloud Run requires app to listen on $PORT
ENV PORT=4000
EXPOSE 4000

# Start run service
CMD ["mix", "phx.server"]
