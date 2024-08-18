# Momo Store aka Пельменная №2

<img width="900" alt="image" src="https://user-images.githubusercontent.com/9394918/167876466-2c530828-d658-4efe-9064-825626cc6db5.png">

## Frontend

-------------------------------
HTML page on nginx

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

------------------------------
Build frontend.

Stage1. Build docker image:

#Use an official Node.js runtime as a parent image

FROM node:16 as build-stage

#Set the working directory

WORKDIR /app

#Copy package.json and package-lock.json

COPY package*.json ./

#Install dependencies

RUN npm install

#Копируем все остальные файлы проекта в текущую рабочую директорию контейнера

COPY . .

#Собираем приложение для production с указанием переменной окружения VUE_APP_API_URL

ENV NODE_ENV=production

#'localhost' should be changed to the resolveable FQDN of a real host where nginx is deployed to be able to proxy requests to backend

ENV VUE_APP_API_URL=http://momo-store.devops-practicum.ru:8081

RUN npm run build

----------------------------------
Stage2: configure nginx:

#Второй этап: настраиваем Nginx для обслуживания статического содержимого

FROM nginx:stable-alpine

ARG VERSION

WORKDIR /app

#Копируем собранное приложение из предыдущего этапа в рабочую директорию Nginx

COPY --from=build-stage /app/dist ./momo-store

#Copy the custom Nginx configuration file

COPY default.conf /etc/nginx/conf.d/default.conf

#Конфигурируем Nginx для обслуживания статического содержимого

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

===========================================================================================

## Backend. Build Go binary in Docker image

-----------------------------
Stage1. Build docker image

#Шаг сборки

FROM golang:1.19-alpine AS builder

#Install gcc for CGO

RUN apk update && apk add --no-cache gcc musl-dev

WORKDIR /app

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o momo-store ./cmd/api

----------------------
Stage2. Run backend as appuser.

#Шаг релиза

FROM alpine:latest

ARG VERSION

WORKDIR /app

COPY --from=builder /app/momo-store /app/momo-store

#COPY --from=builder /app/certs /app/certs

#Добавление непривилегированного пользователя для запуска приложения

RUN addgroup --system appuser && adduser -S -s /bin/false -G appuser appuser -D -H

#Устанавливаем владельца и права доступа на директорию и файлы

RUN chown -R appuser:appuser /app/momo-store

USER appuser

#Порт, на котором будет работать приложение

EXPOSE 8081

#Команда запуска приложения

ENTRYPOINT ["/app/momo-store"]

======================================================================================

## Automatization in Gitlab

---------------------------------
Build, test and release upload are automated in gitlab.

Repository: https://gitlab.praktikum-services.ru/std-026-53/momo-store.git

Docker images are uploaded to docker registry gitlab via crane

Artifacts are created in a separate job and uploaded to a Nexus Raw repository:

Index of /1-0-1449796

Name	Last Modified	Size	Description

Parent Directory

momo-frontend-artifacts-1-0-1449796.tar.gz	Wed Aug 14 16:07:52 UTC 2024	501630	

Index of /1-0-1448758

Name	Last Modified	Size	Description

Parent Directory

backend-artifacts-1-0-1448758.tar.gz	Wed Aug 14 06:56:04 UTC 2024	7011620

=======================================================================================

## Testing:

-----------------------------------
Sonarqube frontend Passed:

https://sonarqube.praktikum-services.ru/dashboard?id=26_AlexLevashov_momo_front

Sonarqube backend Passed:

https://sonarqube.praktikum-services.ru/dashboard?id=26_AlexLevashov_momo_back

-------------------------
Gitlab SAST Passed 

=====================================================================================

## Infrastructure.

----------------------
Deployed a separate Ubuntu VM in YC with Public IP.

-----------------------
Deployed VAULT in docker container via Unit file

created script to unseal vault > login vault > yc iam create-token > put it into vault secrets

added secrets:

vault_token (used in main.tf for terraform)

YC cloud token (used in main.tf for terraform)

-------------------------------
Deployed Minio Local S3 repository

YC object storage is synchronized with it via rclone:

Устанавливаем rclone на ВМ:

$ curl https://rclone.org/install.sh | sudo bash

Настраиваем конфиг rsync для работы с хранилищами:

$ vi ~/.config/rclone/rclone.conf

[yandex-s3]

type = s3

provider = AWS

access_key_id = <>

secret_access_key = <>

region = <>

endpoint = storage.yandexcloud.net

[minio]

type = s3

provider = Minio

env_auth = false

access_key_id = <>

secret_access_key = <>

region = <>

endpoint = http://127.0.0.1:9000

Синхронизируем данные:

rclone sync yandex-s3:alexlevashov-momo-store minio:momo-store

Посмотреть список файлов можно: $ rclone ls <minio или yandex-s3>:<имя бакета>.

Скрипт для синхронизации по cron:

$ crontab -e

0 * * * * rclone sync yandex-s3:<имя бакета в Яндекс Облаке> minio:<имя бакета в MinIO>

Проверяем, что файлы из облачного хранилища Яндекса появились в локальной папке и отображаются в веб-интерфейсе MinIO.

---------------------------------
Purchased domain devops-practicum.ru

----------------------------------
Issued certificates for alev-node1-vm-1.devops-practicum.ru and momo-store.devops-practicum.ru

-------------------------------------------------
K8s cluster was deployed in Yandex Cloud (step-by-step guide C:\DevOps\Практикум\Дипломный проект\YC_Managed_Cluster)

Source repo: https://github.com/geksogen/k8s_install_yandex_cloud_rke/blob/k8s_cluster_install_ya_cloud/README.md

Edited main.tf>valut_token and vault_host are taken to variables.tf from environment variables>vault root token and yc token are taken from vault secrets

Added to variables.tf cloud_id, folder_id, zone, vault_token and vault_host

Nodes are deployed successfully after the changes>terraform init>terraform apply

----------------------------
K8s cluster deployed in Yandex Cloud (step-by-step guide C:\DevOps\Практикум\Дипломный проект\YC_Managed_Cluster)
Attempts to deploy hosted k8s cluster were not successful and LoadBalancer couldn't occupy any Public IP
>deployed Managed service for Kubernetes in YC

## Managed service for Kubernetes

Deployed per documentation: https://yandex.cloud/ru/docs/managed-kubernetes/quickstart

------------------------------
Deployed external Load Balancer per documentation: https://yandex.cloud/ru/docs/managed-kubernetes/operations/create-load-balancer-with-ingress-nginx

Configured port-forwarding for LoadBalancer: 

created values.yml to enable port-forward for backend: https://yandex.cloud/ru/docs/managed-kubernetes/operations/create-load-balancer-with-ingress-nginx#port-forwarding

tcp: {8081: "default/backend:8081"}

portNamePrefix: "momo"

-----------------------------
Installed Loki: https://yandex.cloud/ru/docs/managed-kubernetes/operations/applications/loki

------------------------------
Installed Prometheus Grafana: https://yandex.cloud/ru/docs/managed-kubernetes/operations/applications/prometheus-operator?utm_referrer=https%3A%2F%2Fyandex.cloud%2Fru%2Fdocs%2Fapplication-load-balancer%2Fconcepts%2Fapplication-load-balancer

========================================================================================

## Deploy.

---------------------------------
Created Infrastructure repository: https://gitlab.praktikum-services.ru/std-026-53/momo-infrastructure/

-------------------------------------
Collected pictures in a separate folder

-----------------------------------------------------------
Kubernetes

Performed deploy in Kubernetes via kubectl.

Kubernetes pipeline:

stages:

  - deploy

deploy-kubernetes:

  stage: deploy

  image: docker:24.0.7-alpine3.19

  before_script:

    - apk update && apk add --no-cache docker-cli-compose openssh-client bash curl gettext

    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    - install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    - eval $(ssh-agent -s)

    - echo "$SSH_PRIVATE_KEY"| tr -d '\r' | ssh-add -

    - mkdir -p ~/.ssh

    - chmod 600 ~/.ssh

    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts

    - chmod 644 ~/.ssh/known_hosts

    - mkdir -p ~/.kube

    - echo "$KUBECONFIG_BASE64" | base64 -d >> ~/.kube/config

    - kubectl config use-context momo-store-context

      #- kubectl config use-context k8s-cluster

  script:

    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

    - kubectl apply -f kubernetes/backend/service.yml --namespace default

    - kubectl apply -f kubernetes/backend/vpa.yml --namespace default

    - kubectl apply -f kubernetes/backend/deployment.yml --namespace default

    - kubectl apply -f kubernetes/backend/secrets.yml --namespace default

    - kubectl wait --for=condition=available --timeout=60s deployment/backend --namespace default

    - kubectl apply -f kubernetes/frontend/configmap.yml --namespace default

    - kubectl apply -f kubernetes/frontend/service.yml --namespace default

    - kubectl apply -f kubernetes/frontend/deployment.yml --namespace default

    - kubectl wait --for=condition=available --timeout=60s deployment/frontend --namespace default

    - kubectl apply -f kubernetes/frontend/ingress.yml --namespace default

    - kubectl wait --for=condition=available --timeout=60s deployment/backend --namespace default

  after_script:

    - rm ~/.kube/config

  rules:

    - changes:

      - kubernetes/**/*

  environment:

    name: production/backend

    url: http://momo-store.devops-practicum.ru:80

    auto_stop_in: 1h

  rules:

    - when: manual

-----------------------------------------------------------
Helm

Configured helm chart and performed deploy via helm chart.

.docker/config.json is added as CICD variable as JSON and is used to create docker-config-secret.
(It was not possible to pass CICD variable in helm via base64 encoded CICD variable and use --set)


Helm pipeline:

deploy-helm:

  stage: deploy

  image: alpine/helm:3.9.4

  before_script:

    # Установка репозитория Helm из Nexus

    - helm repo add my-repo ${HELM_REPO} --username ${NEXUS_USER} --password ${NEXUS_PASS}

    - helm repo update

    # Kubectl install

    - apk update && apk add --no-cache curl

    - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    - install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    - mkdir -p ~/.kube

    - echo "$KUBECONFIG_BASE64" | base64 -d >> ~/.kube/config

    - chmod 600 ~/.kube/config

    - kubectl config use-context momo-store-context

    # Убедитесь, что Helm может взаимодействовать с кластером

    - helm repo update

  script:
    
    #remove secret to avoid conflict because of secret already exists

    - kubectl delete secret docker-config-secret --namespace default

    #create docker-config-secret via kubectl>use CICD variable DOCKER_CONFIG_JSON

    - kubectl create secret generic docker-config-secret --namespace default  --from-literal=.dockerconfigjson="$DOCKER_CONFIG_JSON" --type=kubernetes.io/dockerconfigjson

    #install application from the latest version of the helm chart from the Helm repo located in Nexus

    - helm upgrade --install momo-store-chart my-repo/momo-store --atomic --namespace default

  after_script:

    - rm ~/.kube/config

  environment:

    name: production

    url: http://momo-store.devops-practicum.ru:80

    auto_stop_in: 1h

  rules:

    - when: manual

Merged to main after the confirmation Application is successfully deployed via new version of helm chart.

------------------------------------------------------
Helm releases are stored in Nexus repo: http://nexus.praktikum-services.tech/repository/alexlevashov-helm-026/

Index of /momo-store

Name	Last Modified	Size	Description

Parent Directory

0.0.1

0.1.0

0.1.1

Deploy works and application is available: http://momo-store.gitlab-practicum.ru 

=======================================================

## Monitoring

Use utilities available in Yandex Cloud: https://yandex.cloud/ru/docs/managed-kubernetes/metrics

Created Dashboards to monitor pods: https://monitoring.yandex.cloud/folders/b1g3acl1dihgarklvhm3/dashboards/momo-store-dashboard?from=now-1d&to=now&refresh=60000

Alerting in YC






