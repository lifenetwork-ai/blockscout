FROM hexpm/elixir:1.17.3-erlang-27.3.4-alpine-3.21.3 AS builder

ENV MIX_ENV=prod
ENV PORT=4000

WORKDIR /app

RUN apk --no-cache --update add \
  alpine-sdk gmp-dev automake libtool inotify-tools autoconf \
  python3 file gcompat libstdc++ curl ca-certificates git make bash \
  nodejs npm openssl

# Copy toàn bộ source trước để tránh missing deps
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

# Generate self-signed cert
RUN cd apps/block_scout_web && \
    mix phx.gen.cert blockscout blockscout.local && \
    cd -

# Create user and group
ARG BLOCKSCOUT_USER=blockscout
ARG BLOCKSCOUT_GROUP=blockscout
ARG BLOCKSCOUT_UID=10001
ARG BLOCKSCOUT_GID=10001

RUN addgroup -g ${BLOCKSCOUT_GID} ${BLOCKSCOUT_GROUP} && \
    adduser -S -u ${BLOCKSCOUT_UID} -G ${BLOCKSCOUT_GROUP} ${BLOCKSCOUT_USER}

# Change ownership
RUN chown -R ${BLOCKSCOUT_USER}:${BLOCKSCOUT_GROUP} /app

USER ${BLOCKSCOUT_USER}

EXPOSE 4000

CMD ["mix", "phx.server"]
