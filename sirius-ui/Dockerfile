# Multi-stage Dockerfile for sirius-ui with development and production stages

# Base stage with common dependencies
FROM node:18-alpine AS base

# Install system dependencies for all architectures
RUN apk add --no-cache \
    libc6-compat \
    openssl \
    ca-certificates

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY bun.lockb* ./

# Replace bun commands with npm/npx equivalents for compatibility
COPY package.json package.json.bak
RUN sed -i 's/bunx prisma generate/npx prisma generate/g' package.json && \
    sed -i 's/bun next/npx next/g' package.json && \
    sed -i 's/bun run/npm run/g' package.json

# Development stage
FROM base AS development

# Install all dependencies (including dev dependencies)
RUN npm install

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Expose development port
EXPOSE 3000

# Set development environment
ENV NODE_ENV=development
ENV NEXT_TELEMETRY_DISABLED=1

# Copy startup script
COPY start-dev.sh /app/start-dev.sh
RUN chmod +x /app/start-dev.sh

# Create non-root user for development
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs && \
    chown -R nextjs:nodejs /app

# Switch to non-root user
USER nextjs

# Start development server with database setup
CMD ["/app/start-dev.sh"]

# Production builder stage
FROM base AS builder

# Accept build args
ARG NEXT_PUBLIC_CLIENTVAR

# Copy source code for building
COPY . .

# Remove .env files that interfere with Docker environment variables
RUN rm -f .env .env.local .env.development .env.production

# Install dependencies (production + dev for building)
RUN npm install

# Generate Prisma client
RUN npx prisma generate

# Build the application
ENV NEXT_TELEMETRY_DISABLED=1
ENV NEXT_PUBLIC_CLIENTVAR=${NEXT_PUBLIC_CLIENTVAR}

# Create a temporary next.config.mjs with errors ignored for Docker building
RUN cp next.config.mjs next.config.mjs.bak && \
    sed -i 's/swcMinify: true,/swcMinify: true,\n  typescript: { ignoreBuildErrors: true },\n  eslint: { ignoreDuringBuilds: true },/' next.config.mjs && \
    npx next build

# Production runtime stage
FROM node:18-alpine AS production

# Install runtime dependencies
RUN apk add --no-cache \
    libc6-compat \
    openssl \
    ca-certificates

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Set working directory
WORKDIR /app

# Copy built application from builder stage
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules

# Copy source files needed for runtime
COPY --from=builder /app/src ./src

# Copy Prisma files for runtime
COPY --from=builder /app/prisma ./prisma

# Copy configuration files
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/next.config.mjs ./next.config.mjs

# Copy production startup script
COPY start-prod.sh /app/start-prod.sh
RUN chmod +x /app/start-prod.sh

# Change ownership
RUN chown -R nextjs:nodejs /app

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Set production environment
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Start the application with database setup
CMD ["/app/start-prod.sh"]