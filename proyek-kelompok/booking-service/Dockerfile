FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev          # lapisan ini di-cache selama package*.json tidak berubah
COPY . .
USER node                      # jangan jalankan sebagai root
EXPOSE 3001
CMD ["node", "server.js"]
