# syntax=docker/dockerfile:1.6

# ---------- Build stage ----------
FROM node:24.14.1-bookworm-slim AS builder

WORKDIR /app

RUN corepack enable

COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

COPY . .

ARG ARGO_VERSION=latest
ENV ARGO_VERSION=${ARGO_VERSION}
ENV NODE_ENV=production
ENV NODE_OPTIONS="--max-old-space-size=4096"

RUN mkdir -p dist && pnpm build

# ---------- Runtime stage ----------
FROM nginx:1.27-alpine

RUN apk add --no-cache gettext

COPY --from=builder /app/dist/app /usr/share/nginx/html

COPY <<'EOF' /etc/nginx/templates/default.conf.template
server {
    listen       ${PORT};
    server_name  _;

    root   /usr/share/nginx/html;
    index  index.html;

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Argo CD API + auth + swagger
    location ~ ^/(api|auth|extensions|swagger-ui|swagger\.json) {
        proxy_pass ${ARGOCD_API_URL};
        proxy_http_version 1.1;
        proxy_set_header Host              $proxy_host;
        proxy_ssl_server_name on;
        proxy_ssl_verify on;
        proxy_read_timeout 1h;
        proxy_send_timeout 1h;
    }

    # Pod/container terminal (WebSocket)
    location /terminal {
        proxy_pass ${ARGOCD_API_URL};
        proxy_http_version 1.1;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $proxy_host;
        proxy_ssl_server_name on;
        proxy_ssl_verify on;
        proxy_read_timeout 1h;
        proxy_send_timeout 1h;
    }
}
EOF

ENV PORT=26519
ENV ARGOCD_API_URL=http://argocd-server:80

EXPOSE 26519

CMD ["nginx", "-g", "daemon off;"]
