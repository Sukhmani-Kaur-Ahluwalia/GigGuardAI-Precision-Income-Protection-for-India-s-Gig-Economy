FROM node:20-slim AS builder

WORKDIR /app

RUN corepack enable

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml tsconfig.json tsconfig.base.json .npmrc ./
COPY artifacts ./artifacts
COPY lib ./lib
COPY attached_assets ./attached_assets
COPY scripts ./scripts

RUN pnpm install --frozen-lockfile

# The Vite app requires these variables even for production builds.
ENV PORT=4173
ENV BASE_PATH=/

RUN pnpm run build:web
RUN pnpm run build:api

FROM node:20-slim AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=8080

COPY --from=builder /app /app

CMD ["node", "--enable-source-maps", "artifacts/api-server/dist/index.mjs"]
