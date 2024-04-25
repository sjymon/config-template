# Install dependencies only when needed
FROM node:18-alpine AS deps

# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat

WORKDIR /app
# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Rebuild the source code only when needed
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js collects anonymous telemetry data about general usage, which we opt out from
# https://nextjs.org/telemetry
ENV NEXT_TELEMETRY_DISABLED 1

ARG ENVIRONMENT=dev

RUN \
  if [ -f yarn.lock ]; then yarn run build:${ENVIRONMENT}; \
  elif [ -f package-lock.json ]; then npm run build:${ENVIRONMENT}; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm run build:${ENVIRONMENT}; \
  else echo "Lockfile not found." && exit 1; \
  fi

# Production image, copy all the files and run next
FROM node:18-alpine AS runner
WORKDIR /app

# Disable telemetry during runtime
ENV NEXT_TELEMETRY_DISABLED 1

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

# You only need to copy next.config.js if you are NOT using the default configuration
COPY --from=builder /app/next.config.mjs ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/node_modules ./node_modules

EXPOSE 3000

ENV PORT 3000

CMD ["npm", "run", "start"]
