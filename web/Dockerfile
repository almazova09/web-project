FROM node:18 AS builder

WORKDIR /usr/src/app

COPY package.json package-lock.json* ./
RUN npm install

COPY app.js ./
COPY bin ./bin
COPY routes ./routes
COPY views ./views
COPY public ./public

FROM node:18-slim
WORKDIR /usr/src/app

COPY --from=builder /usr/src/app .

EXPOSE 3000
CMD ["npm","start"]
