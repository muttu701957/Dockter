#BUILD STAGE 
FROM node:20-alpine AS builder

#SET WORKING DIR
WORKDIR /app

COPY package.json ./
RUN npm install --legacy-peer-deps

COPY . .

ARG VITE_BACKEND_URL=https://service.zeventro.com
ENV VITE_BACKEND_URL=$VITE_BACKEND_URL

RUN npm run build

FROM nginx:1.27-alpine AS runner

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
