FROM node:18-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm config set strict-ssl false && npm ci --legacy-peer-deps

COPY . .

RUN NODE_OPTIONS=--openssl-legacy-provider npm run build:web

FROM nginx:alpine

COPY --from=build --chown=nginx:nginx /app/dist/web /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
