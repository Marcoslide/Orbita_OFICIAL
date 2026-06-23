# ---------- Órbita API — Dockerfile de produção (multi-stage) ----------
# Estágio 1: build (instala devDeps, gera Prisma Client, compila TS -> dist)
FROM node:20-slim AS builder
WORKDIR /app

# Dependências do sistema necessárias ao Prisma (openssl)
RUN apt-get update -y && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*

# Instala dependências com cache de layer (copia manifests primeiro)
COPY package*.json ./
COPY prisma ./prisma
RUN npm ci

# Copia o restante do código e compila
COPY . .
RUN npx prisma generate
RUN npm run build

# Remove devDependencies para a imagem final ficar enxuta
RUN npm prune --omit=dev

# ---------- Estágio 2: runtime (imagem mínima, só o necessário) ----------
FROM node:20-slim AS runner
WORKDIR /app
ENV NODE_ENV=production

RUN apt-get update -y && apt-get install -y openssl curl && rm -rf /var/lib/apt/lists/*

# Usuário não-root
RUN groupadd -r orbita && useradd -r -g orbita orbita

# Copia artefatos do builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/package*.json ./

# Prisma Client gerado (engine) precisa estar presente
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

USER orbita
EXPOSE 3333

# Healthcheck interno (liveness) — o orquestrador também pode usar /v1/health/ready
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD curl -fsS http://localhost:3333/v1/health/live || exit 1

# Em produção, migrations são aplicadas pelo script de deploy (migrate.sh),
# não no entrypoint, para evitar corrida entre réplicas. Aqui só sobe a API.
CMD ["node", "dist/main.js"]
