#frontend/Dockerfile

FROM node:18 as build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

ENV REACT_APP_BACKEND_URL=/api

RUN npm run build

FROM nginx:1.25
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
