FROM node:10

RUN mkdir -p /opt/racecourse && chown -R node:node /opt/racecourse

WORKDIR /opt/racecourse

COPY . .

COPY --chown=node:node . .

USER node

EXPOSE 3000

WORKDIR /opt/racecourse/client

RUN npm install

RUN npm run build

WORKDIR /opt/racecourse/server

RUN npm install

CMD ["node", "server.js"]
