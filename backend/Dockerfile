# ---------- BUILD STAGE ------
FROM node:20 AS builder

WORKDIR /app

COPY package*.json ./

RUN npm install --legacy-peer-deps

COPY . .

# ---------- PRODUCTION STAGE ----------
FROM node:20-slim

WORKDIR /app

COPY package*.json ./

# Install only production dependencies
RUN npm install --omit=dev

# Copy application source
COPY --from=builder /app .

EXPOSE 4000

CMD ["npm", "start"]
