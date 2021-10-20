# Nodejs项目的Dockerfile模版

## 基于Yarn

```Dockerfile
FROM node:14.17.6-alpine3.14 as build

# 取消对https的证书限制
# ENV SELF_SIGNED_CERT_IN_CHAIN=true
# ENV NODE_TLS_REJECT_UNAUTHORIZED=0
# RUN npm config set strict-ssl false
# RUN yarn config set strict-ssl false

RUN apk update && apk add bash

COPY package.json yarn.lock .npmrc /tmp/
RUN cd /tmp && yarn install --frozen-lockfile --non-interactive
RUN mkdir -p /app && cp -a /tmp/node_modules /app/

WORKDIR /app
COPY . /app
RUN yarn build


FROM build AS ci
WORKDIR /app
RUN yarn run test:ci && yarn install --frozen-lockfile --non-interactive --production

FROM node:14.17.6-alpine3.14 as release

RUN mkdir -p /app && \
	addgroup -S daminggroup && \
	adduser -S -h /app -G daminggroup daminguser && \
	chown -R daminguser:daminggroup /app

WORKDIR /app
COPY --from=ci --chown=daminguser:daminggroup /app/dist ./dist
COPY --from=ci --chown=daminguser:daminggroup /app/node_modules ./node_modules
COPY --from=ci --chown=daminguser:daminggroup /app/package.json .

HEALTHCHECK --interval=30s --timeout=30s CMD curl -f http://localhost:3978/ping || exit 1
EXPOSE 3978
CMD ["sh", "-c", "node dist/index.js"]
```

## 基于NPM

```Dockerfile
FROM node:14.17.6-alpine3.14 as build

# pwc network limit
# ENV SELF_SIGNED_CERT_IN_CHAIN=true
# ENV NODE_TLS_REJECT_UNAUTHORIZED=0
# RUN npm config set strict-ssl false
# RUN yarn config set strict-ssl false

RUN apk update && apk add bash && npm install -g npm

COPY package.json package-lock.json .npmrc /tmp/
RUN cd /tmp && npm install
RUN mkdir -p /app && cp -a /tmp/node_modules /app/

WORKDIR /app
COPY . /app
RUN npm run build


FROM build AS ci
WORKDIR /app
RUN npm run test:ci && npm install --production --ignore-scripts true

FROM node:14.17.6-alpine3.14 as release

RUN mkdir -p /app && \
	addgroup -S daminggroup && \
	adduser -S -h /app -G daminggroup daminguser && \
	chown -R daminguser:daminggroup /app

WORKDIR /app
COPY --from=ci --chown=daminguser:daminggroup /app/dist ./dist
COPY --from=ci --chown=daminguser:daminggroup /app/node_modules ./node_modules
COPY --from=ci --chown=daminguser:daminggroup /app/package.json .

HEALTHCHECK --interval=30s --timeout=30s CMD curl -f http://localhost:3978/ping || exit 1
EXPOSE 3978
CMD ["sh", "-c", "node dist/index.js"]
```