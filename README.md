# Momo Store aka Пельменная №2

<img width="900" alt="image" src="https://user-images.githubusercontent.com/9394918/167876466-2c530828-d658-4efe-9064-825626cc6db5.png">

## Frontend
HTML page via nginx
default.conf for nginx:
server {
    listen 80;

    root /app/momo-store;
    index index.html;

    location /momo-store/ {
        alias /app/momo-store/;
        try_files $uri $uri/ /momo-store/index.html;
    }
    location /api {
        proxy_pass http://backend:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Stage1. Build docker image:
# Use an official Node.js runtime as a parent image
FROM node:16 as build-stage

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Копируем все остальные файлы проекта в текущую рабочую директорию контейнера
COPY . .

#  Собираем приложение для production с указанием переменной окружения VUE_APP_API_URL
ENV NODE_ENV=production
# 'localhost' should be changed to the resolveable FQDN of a real host where nginx is deployed to be able to proxy requests to backend
ENV VUE_APP_API_URL=http://momo-store.devops-practicum.ru:8081
RUN npm run build

----------------------------------
Stage2: configure nginx:
# Второй этап: настраиваем Nginx для обслуживания статического содержимого
FROM nginx:stable-alpine
ARG VERSION
WORKDIR /app

# Копируем собранное приложение из предыдущего этапа в рабочую директорию Nginx
COPY --from=build-stage /app/dist ./momo-store

# Copy the custom Nginx configuration file
COPY default.conf /etc/nginx/conf.d/default.conf

# Конфигурируем Nginx для обслуживания статического содержимого
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

===========================================================================================
## Backend. Build Go binary in Docker image
Stage1. Build docker image
# Шаг сборки
FROM golang:1.19-alpine AS builder

# Install gcc for CGO
RUN apk update && apk add --no-cache gcc musl-dev

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

#RUN go test -v ./... ##separately tested in pipeline

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o momo-store ./cmd/api

----------------------
Stage2. Run backend as appuser.
# Шаг релиза
FROM alpine:latest

ARG VERSION

WORKDIR /app

COPY --from=builder /app/momo-store /app/momo-store

#COPY --from=builder /app/certs /app/certs

# Добавление непривилегированного пользователя для запуска приложения
RUN addgroup --system appuser && adduser -S -s /bin/false -G appuser appuser -D -H

# Устанавливаем владельца и права доступа на директорию и файлы
RUN chown -R appuser:appuser /app/momo-store

USER appuser

# Порт, на котором будет работать приложение
EXPOSE 8081

# Команда запуска приложения
ENTRYPOINT ["/app/momo-store"]

======================================================================================

Automatization in gitlab.
https://gitlab.praktikum-services.ru/std-026-53/momo-store.git


