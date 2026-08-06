---
title: GitLab CI: How to Build Docker Images in Docker
url: https://devopstales.github.io/devops/gitlab-ci-docker-bild/
date: 2023-02-06
keywords: devops, devops tools, IAC, gitlab, ci/cd, Continuous Deployments, Continuous Integration, docker build, Gitlab package registry
---


One of the most common use case is to build a Docker image with Gitlab. In this post I will show you how to set up Docker builds in CI.

<!--more-->

### Shell Executor

The easiest way to build a docker image is by using the Shell executor. For this you need a standard linux based Gitlab Runner with Docker installed on it. Then the gitlab runner user needs to be in the `docker group` to execute docker commands.

To build a Docker image you mast store the `Dockerfile` in your repository. Here is the `.gitlab-ci.yaml` for this build:

```yaml
stages:
  - build

docker_build:
  stage: build
  script:
    - docker build -t example.com/example-image:latest .
    - docker push example.com/example-image:latest
  tags: 
    - docker
```

To push an image to a docker registry you need to authenticate. Luckily Gitlab has it's own docker registry and automatically gives access to the runners. So you can use the built-in variables for this:

```yaml
stages:
  - build

docker_build:
  stage: build
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:latest .
    - docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:latest
  tags: 
    - docker
```

The next problem is to identify your images, which image belongs to which build.

```yaml
stages:
  - build

docker_build:
  stage: build
  script: |
    docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    if [[ "$CI_BUILD_REF_NAME" == "master" ]]; then
      DOCKER_TAG="latest"
    else
      DOCKER_TAG=$CI_COMMIT_REF_SLUG
    fi
    docker build -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:latest .
    docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:latest
    echo $DOCKER_TAG
  tags: 
    - docker
```

It is a best practice to use the commit hash as the tag. For release you can use the git release tag as tag:

```yaml
stages:
  - build

docker_build:
  stage: build
  script: |
    docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    if [[ "$CI_BUILD_REF_NAME" == "master" ]]; then
      DOCKER_TAG="latest"
    else
      DOCKER_TAG=$CI_COMMIT_SHA
    fi
    docker build -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:$DOCKER_TAG .
    docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:$DOCKER_TAG
    echo $DOCKER_TAG
  tags: 
    - docker

docker_release:
  stage: build
  script: |
    docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    docker build -t $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:$CI_COMMIT_TAG .
    docker push $CI_REGISTRY/$CI_PROJECT_NAMESPACE/$CI_PROJECT_NAME/example-image:$CI_COMMIT_TAG
    echo $CI_COMMIT_TAG
  tags: 
    - docker
  only:
    - master
    - tags
    - /^release-.*$/
```

Usually you build your app in a separate job dan put your artifact in the docker image, but multistage docker build you can do thi in tha same job. For this you need t edit your `Dockerfile` to have an application build.

```yaml
FROM node:8 AS builder

ADD package*.json /app/
WORKDIR /app
RUN npm install

ADD . /app
RUN npm run-script build

FROM nginx
COPY --from=builder /app/dist/* /usr/share/nginx/html/
COPY --from=builder /app/dist/assets /usr/share/nginx/html/assets
```

```yaml
stages:
  - build

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_PIPELINE_ID

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

docker_build:
  stage: build
  script:
    # Here we try to download a Docker image with the :builder tag (which contains all the layers from the last build)
    - docker pull $CI_REGISTRY_IMAGE:builder || true
    # It will run the first part of the Dockerfile. Also we tell Docker to use the cache layers from the previous build.
    - docker build --pull --cache-from $CI_REGISTRY_IMAGE:builder --target builder -t $CI_REGISTRY_IMAGE:builder .
    - docker build --pull --cache-from $CI_REGISTRY_IMAGE:builder --cache-from $IMAGE_TAG -t $IMAGE_TAG -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:builder
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $IMAGE_TAG
  tags: 
    - docker
```

### Building With the Docker Executor

You can use docker as an environment to run all the scripts to provide a completely clean environment for each job. For this you need to use gitlab runner in a so called Docker-in-Docker (DinD) mode. In a normal docker environment yo can not use docker commands in the container. When you use the docker cli it needs to connect to the docker engine unix socket. To do this you need to mount the docker socket in the container and run as privileged mode.


Register your runner in DinD mode:

```bash
sudo gitlab-runner register -n 
  --url https://example.com 
  --registration-token $GITLAB_REGISTRATION_TOKEN 
  --executor docker 
  --description "Docker Runner" 
  --docker-image "docker:20.10" 
  --docker-volumes "/certs/client" 
  --docker-privileged
```

 When you configure CI/CD, you specify an image, which is used to create the container where your jobs run. To specify this image, you use the `image` keyword. ou can specify an additional image by using the `services` keyword. Within your CI pipeline, add the `docker:dind` image as a `service`. This gives a separate image to communicate with the docker engine.

```yaml
services:
  - docker:dind

docker_build:
  stage: build
  image: docker:latest
  script:
    - docker build -t example-image:latest .
```


