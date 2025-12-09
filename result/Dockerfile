FROM node:18-alpine
WORKDIR /app
RUN apk add --no-cache tini
COPY package*.json ./
RUN npm ci && npm cache clean --force
COPY . .
EXPOSE 80
ENTRYPOINT ["/sbin/tini", "--"]
# utilisateur non root
COPY --chown=node:node . .
USER node
CMD ["node", "server.js"]