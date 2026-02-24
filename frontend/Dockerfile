# ================================
# Stage 1: Build Angular app
# ================================
FROM node:20-alpine AS builder
WORKDIR /app

# Copy dependency manifests
COPY package.json package-lock.json* ./

# Install all deps (including devDependencies for build)
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build -- --configuration production

# ================================
# Stage 2: Serve with nginx
# ================================
FROM nginx:stable-alpine AS production
# Remove default nginx static content
RUN rm -rf /usr/share/nginx/html/*

# Nginx config for Angular SPA routing
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Copy built app from builder (Angular output path from angular.json)
COPY --from=builder /app/dist/angular-15-crud /usr/share/nginx/html

# Expose port 80 (override with docker run -p if needed)
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
