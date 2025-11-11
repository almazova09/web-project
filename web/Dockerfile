FROM node:18 AS builder

WORKDIR /usr/src/app

COPY package.json .

RUN npm install

COPY . .

# next stage
FROM node:18-slim

WORKDIR /usr/src/app

COPY --from=builder /usr/src/app .

ENV PORT=3000 \
    API_HOST=http://34.122.84.206:3001

EXPOSE 3000

CMD ["npm","start"]

