FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
# Lapisan ini di-cache selama package*.json tidak berubah
RUN npm ci --omit=dev
COPY . .
# Jangan jalankan sebagai root
USER node
EXPOSE 3000
CMD ["node", "server.js"]
