# Frontend build stage
FROM node:22-alpine AS frontend-builder
RUN corepack enable && corepack prepare pnpm@10.28.2 --activate
WORKDIR /app/frontend
COPY clients/frontend/package.json clients/frontend/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY clients/frontend/ ./
RUN pnpm run build

# --- FIX: Compile Rust components directly during the build ---
FROM rust:1.80-slim AS rust-builder

# Install necessary build dependencies for Rust compilation
RUN apt-get update && apt-get install -y --no-install-recommends \
    pkg-config \
    libssl-dev \
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
# Copy the entire source code into the builder stage
COPY . .

# Compile the application binaries in release mode
RUN cargo build --release

# Copy the compiled binaries to the expected locations for the runtime stage
RUN cp target/release/mediafusion-api /mediafusion-api && \
    cp target/release/mediafusion-worker /mediafusion-worker


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

# Pull the binaries we just built from the rust-builder stage
COPY --from=rust-builder --chown=mediafusion:mediafusion /mediafusion-api    /usr/local/bin/mediafusion-api
COPY --from=rust-builder --chown=mediafusion:mediafusion /mediafusion-worker /usr/local/bin/mediafusion-worker

COPY --chown=mediafusion:mediafusion . /mediafusion

COPY --from=frontend-builder --chown=mediafusion:mediafusion \
    /app/frontend/dist /mediafusion/clients/frontend/dist

ARG VERSION
ENV VERSION=${VERSION}

USER mediafusion

EXPOSE 8000

# Default: run the API server.
CMD ["/usr/local/bin/mediafusion-api"]
