FROM node:20-alpine AS deps
WORKDIR /app

COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

COPY --from=deps /app/node_modules ./node_modules
COPY server.js ./server.js
COPY public ./public
COPY package*.json ./

EXPOSE 3000
CMD ["node", "server.js"]
