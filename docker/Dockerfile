# ============================
# Stage 1: Build dependencies
# ============================
FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3 AS builder-deps

WORKDIR /app

RUN apk --no-cache --update add \
  alpine-sdk gmp-dev automake libtool inotify-tools autoconf python3 \
  file gcompat libstdc++ curl ca-certificates git make bash

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

RUN mix local.hex --force
RUN mix do deps.get, local.rebar --force, deps.compile --skip-umbrella-children

# Copy full source
COPY config ./config
COPY rel ./rel
COPY apps ./apps

# ============================
# Stage 2: Build release
# ============================
FROM builder-deps AS builder

# Custom ENV flags
ENV DISABLE_WEBAPP=true
ENV ADMIN_PANEL_ENABLED=false
ARG DISABLE_API
ENV DISABLE_API=${DISABLE_API}
ARG API_V1_READ_METHODS_DISABLED
ENV API_V1_READ_METHODS_DISABLED=${API_V1_READ_METHODS_DISABLED}
ARG API_V1_WRITE_METHODS_DISABLED
ENV API_V1_WRITE_METHODS_DISABLED=${API_V1_WRITE_METHODS_DISABLED}
ARG CHAIN_TYPE
ENV CHAIN_TYPE=${CHAIN_TYPE}
ARG BRIDGED_TOKENS_ENABLED
ENV BRIDGED_TOKENS_ENABLED=${BRIDGED_TOKENS_ENABLED}
ARG API_GRAPHQL_MAX_COMPLEXITY
ENV API_GRAPHQL_MAX_COMPLEXITY=${API_GRAPHQL_MAX_COMPLEXITY}

RUN mix compile

RUN mkdir -p /opt/release && \
  mix release blockscout && \
  mv _build/prod/rel/blockscout /opt/release

# ============================
# Stage 3: Final container
# ============================
FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3

WORKDIR /app

# Add user for security
ARG BLOCKSCOUT_USER=blockscout
ARG BLOCKSCOUT_GROUP=blockscout
ARG BLOCKSCOUT_UID=10001
ARG BLOCKSCOUT_GID=10001

RUN apk --no-cache --update add jq curl && \
  addgroup --system --gid ${BLOCKSCOUT_GID} ${BLOCKSCOUT_GROUP} && \
  adduser --system --uid ${BLOCKSCOUT_UID} --ingroup ${BLOCKSCOUT_GROUP} --disabled-password ${BLOCKSCOUT_USER}

# Environment
ENV DISABLE_WEBAPP=true
ENV ADMIN_PANEL_ENABLED=false
ARG DISABLE_API
ENV DISABLE_API=${DISABLE_API}
ARG API_V1_READ_METHODS_DISABLED
ENV API_V1_READ_METHODS_DISABLED=${API_V1_READ_METHODS_DISABLED}
ARG API_V1_WRITE_METHODS_DISABLED
ENV API_V1_WRITE_METHODS_DISABLED=${API_V1_WRITE_METHODS_DISABLED}
ARG CHAIN_TYPE
ENV CHAIN_TYPE=${CHAIN_TYPE}
ARG BRIDGED_TOKENS_ENABLED
ENV BRIDGED_TOKENS_ENABLED=${BRIDGED_TOKENS_ENABLED}
ARG API_GRAPHQL_MAX_COMPLEXITY
ENV API_GRAPHQL_MAX_COMPLEXITY=${API_GRAPHQL_MAX_COMPLEXITY}

# Fixed version for compatibility with runtime.exs
ARG RELEASE_VERSION=8.1.1
ENV RELEASE_VERSION=${RELEASE_VERSION}

COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /opt/release/blockscout .
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/config_helper.exs ./config/config_helper.exs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/config_helper.exs /app/releases/${RELEASE_VERSION}/config_helper.exs
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app/config/assets/precompiles-arbitrum.json ./config/assets/precompiles-arbitrum.json

# Runtime folders
RUN mkdir dets temp && chown -R ${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app

USER ${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP}

# Start the release (IMPORTANT for Cloud Run)
CMD ["./bin/blockscout", "start"]
