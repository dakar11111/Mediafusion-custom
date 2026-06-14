# Frontend build stage
FROM node:22-alpine AS frontend-builder
RUN corepack enable && corepack prepare pnpm@10.28.2 --activate
WORKDIR /app/frontend
COPY clients/frontend/package.json clients/frontend/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY clients/frontend/ ./
RUN pnpm run build

# --- Rust Builder Stage ---
FROM rust:1.91-slim AS rust-builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    pkg-config \
    libssl-dev \
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

# Move into the Rust workspace directory before compiling
WORKDIR /app/backend/rust
RUN cargo build --release

# --- FIX: Copy out from the correct central target location ---
RUN cp /app/backend/target/release/mediafusion-api /mediafusion-api && \
    cp /app/backend/target/release/mediafusion-worker /mediafusion-worker


# ── Single runtime image (API default, worker via CMD override) ───────────────
FROM debian:trixie-slim AS runtime

WORKDIR /mediafusion

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    ca-certificates \
    curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN groupadd -r mediafusion && \
    useradd --no-log-init -r -g mediafusion -m -d /home/mediafusion mediafusion

COPY --from=rust-builder /mediafusion-api    /usr/local/bin/mediafusion-api
COPY --from=rust-builder /mediafusion-worker /usr/local/bin/mediafusion-worker

COPY --chown=mediafusion:mediafusion . /mediafusion

COPY --from=frontend-builder --chown=mediafusion:mediafusion \
    /app/frontend/dist /mediafusion/clients/frontend/dist

ARG VERSION
ENV VERSION=${VERSION}

USER mediafusion

EXPOSE 8000

CMD ["/usr/local/bin/mediafusion-api"]
