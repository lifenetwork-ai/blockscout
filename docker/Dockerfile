# =======================
# Build Stage
# =======================
FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3 AS builder

ENV MIX_ENV=prod
ENV PORT=4000
WORKDIR /app

# Install dependencies
RUN apk --no-cache --update add \
  alpine-sdk gmp-dev automake libtool inotify-tools autoconf \
  python3 file gcompat libstdc++ curl ca-certificates git make bash \
  nodejs npm openssl

# Copy source
COPY . .

# Install Elixir deps
RUN mix local.hex --force && \
    mix local.rebar --force && \
    mix deps.get && \
    mix deps.compile && \
    mix compile

# Build frontend assets
RUN cd apps/block_scout_web/assets && \
    npm install && \
    node_modules/webpack/bin/webpack.js --mode production && \
    cd - && \
    cd apps/explorer && \
    npm install && \
    cd - && \
    mix phx.digest

# Generate TLS cert
RUN cd apps/block_scout_web && \
    mix phx.gen.cert blockscout blockscout.local && \
    cd -

# Build release & include config_helper.exs in custom config folder
RUN mix release blockscout && \
    RELEASE_VERSION=8.1.1 && \
    mkdir -p /opt/release/blockscout/config && \
    cp -r _build/prod/rel/blockscout /opt/release/ && \
    cp config/config_helper.exs /opt/release/blockscout/config/ && \
    mkdir -p /opt/release/blockscout/releases/$RELEASE_VERSION && \
    cp config/config_helper.exs /opt/release/blockscout/releases/$RELEASE_VERSION/config_helper.exs

# =======================
# Runtime Stage
# =======================
FROM alpine:3.21

ENV LANG=C.UTF-8
ENV PORT=4000
WORKDIR /app

# Runtime deps
RUN apk --no-cache --update add openssl ncurses libstdc++ curl

ARG BLOCKSCOUT_USER=blockscout
ARG BLOCKSCOUT_GROUP=blockscout
ARG BLOCKSCOUT_UID=10001
ARG BLOCKSCOUT_GID=10001

RUN addgroup -g ${BLOCKSCOUT_GID} ${BLOCKSCOUT_GROUP} && \
    adduser -S -u ${BLOCKSCOUT_UID} -G ${BLOCKSCOUT_GROUP} ${BLOCKSCOUT_USER}

# Copy full release, including config
COPY --from=builder --chown=${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /opt/release/blockscout ./
COPY --from=builder /opt/release/blockscout/config/config_helper.exs ./config/config_helper.exs

USER ${BLOCKSCOUT_USER}
EXPOSE 4000

CMD ["./bin/blockscout", "start"]
