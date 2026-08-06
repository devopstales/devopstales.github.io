---
title: GitLab CI: How to Build Docker Images in Kubernetes
url: https://devopstales.github.io/devops/gitlab-ci-docker-bild-k8s/
date: 2023-02-10
keywords: devops, devops tools, IAC, gitlab, ci/cd, Continuous Deployments, Continuous Integration, docker build, Gitlab package registry, kaniko
---


One of the most common use case is to build a Docker image with Gitlab. In a [previous post]() we used dedicated docker runners for this job. But howe can we build images in a Kubernetes runner? In this post we well se this.

<!--more-->

Using docker engine to build a container is a good solution but in some case can be unsafe. With the privilege to use a docker command you can create or delete any container. With the deprecation of dockershim in kubernetes this option is no longer usable. You can use multiple tools for this but in this case I will use `Kaniko`.

### What is Kaniko?

`kaniko` builds container images from a `Dockerfile` inside a container or Kubernetes cluster. It doesn’t depend on a Docker daemon, and it executes each `Dockerfile` command completely in userspace. This  allows you to build container images in environments that can’t easily or securely run a Docker daemon — such as standard Kubernetes clusters.

### Build images with Kaniko

To build docker images with Kaniko in a Kubernetes Gitlab runner you just simply use its image as the image for the build job:

```yaml
stages:
- build
- deploy_to_kubernetes

variables:
  CACHE_TTL: 2190h0m0s # three months
  IMAGE_NAME: app

before_script:
  - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"},\"$CI_DEPENDENCY_PROXY_SERVER\":{\"auth\":\"$(printf "%s:%s" ${CI_DEPENDENCY_PROXY_USER} "${CI_DEPENDENCY_PROXY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json

build-backend:
  stage: build
  tags:
    - kubernetes
  image:
    name: gcr.io/kaniko-project/executor:v1.9.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor \
      --context "${CI_PROJECT_DIR}" \
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
      --destination "${CI_REGISTRY_IMAGE}/${IMAGE_NAME}:${CI_COMMIT_TAG}" \
      --cache-repo ${CONTAINER_REGISTRY}/${IMAGE_NAME} \
      --cache=true \
      --cache-ttl $CACHE_TTL
```

GitLab provides a functionality to cache commonly used images. It helps you stay within Docker Hub’s rate limits. This will also improve the performance of your builds. To enable Dependency Proxy Go to your project's parent group `Settings > Packages & Registries > Dependency Proxy`

```yaml
stages:
- build
- deploy_to_kubernetes

variables:
  CACHE_TTL: 2190h0m0s # three months
  IMAGE_NAME: app

before_script:
  - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"},\"$CI_DEPENDENCY_PROXY_SERVER\":{\"auth\":\"$(printf "%s:%s" ${CI_DEPENDENCY_PROXY_USER} "${CI_DEPENDENCY_PROXY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json

build-backend:
  stage: build
  tags:
    - kubernetes
  image:
    name: $CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX/executor:v1.9.0-debug
    entrypoint: [""]
  script:
    - /kaniko/executor \
      --context "${CI_PROJECT_DIR}" \
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
      --destination "${CI_REGISTRY_IMAGE}/${IMAGE_NAME}:${CI_COMMIT_TAG}" \
      --registry-mirror "$CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX" \
      --cache-repo ${CONTAINER_REGISTRY}/${IMAGE_NAME} \
      --cache=true \
      --cache-ttl $CACHE_TTL
```
