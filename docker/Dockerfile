# Node.js version (overridable via build-arg from CI)
ARG NODE_VERSION=24

# Stage 1: Build stage
FROM node:${NODE_VERSION}-alpine AS build

USER root

# Install build dependencies
RUN apk add --no-cache libc6-compat git python3 py3-pip make g++ build-base cairo-dev pango-dev && \
    npm install -g pnpm@10.26.0

# Skip downloading Chrome for Puppeteer (saves build time)
ENV PUPPETEER_SKIP_DOWNLOAD=true

ENV NODE_OPTIONS=--max-old-space-size=8192

WORKDIR /usr/src/flowise

# Build Flowise from this checkout using pnpm-lock.yaml, the same dependency tree
# the root Dockerfile and CI use. Installing the published package instead
# (`npm install -g flowise`) resolved every dependency afresh at build time and
# ignored the overrides in packages/server/package.json, so newer transitive
# releases could break the image (e.g. better-sqlite3@9 pulled in via mem0ai does
# not build on Node 24).
COPY . .

RUN pnpm install --frozen-lockfile && \
    pnpm build:docker

# Stage 2: Runtime stage
FROM node:${NODE_VERSION}-alpine

# Install runtime dependencies
RUN apk add --no-cache libc6-compat chromium git python3 py3-pip make g++ build-base cairo-dev pango-dev curl

# Set the environment variable for Puppeteer to find Chromium
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser

# Copy Flowise from the build stage, owned by the node user (node:*-alpine already has this user)
COPY --from=build --chown=node:node /usr/src/flowise /usr/src/flowise

# Keep the `flowise` command on PATH, as the npm-installed image had it
RUN ln -s /usr/src/flowise/packages/server/bin/run /usr/local/bin/flowise

# Switch to non-root user
USER node

ENTRYPOINT ["flowise", "start"]
