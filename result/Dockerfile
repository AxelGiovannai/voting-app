FROM node:18-alpine
WORKDIR /app
RUN apk add --no-cache tini
COPY package*.json ./
RUN npm ci && npm cache clean --force
COPY . .
EXPOSE 80
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]