# ================================
# Stage 1: Install dependencies
# ================================
FROM node:20-alpine AS deps
WORKDIR /app

# Copy dependency manifests for reproducible installs
COPY package.json package-lock.json* ./

# Install production dependencies only (no devDependencies)
RUN npm ci --omit=dev

# ================================
# Stage 2: Production image
# ================================
FROM node:20-alpine AS production
WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy production node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy application source
COPY server.js ./
COPY app ./app

# Own files as nodejs user
RUN chown -R nodejs:nodejs /app

USER nodejs

ENV NODE_ENV=production


EXPOSE 8080

CMD ["node", "server.js"]
