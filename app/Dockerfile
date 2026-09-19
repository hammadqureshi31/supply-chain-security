FROM node:24-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY server.js .

# npm/corepack/yarn are build-time tooling and are not needed at runtime.
RUN rm -rf \
    /usr/local/lib/node_modules/npm \
    /usr/local/lib/node_modules/corepack \
    /opt/yarn-v1.22.22

USER node
EXPOSE 3000
CMD ["node", "server.js"]

