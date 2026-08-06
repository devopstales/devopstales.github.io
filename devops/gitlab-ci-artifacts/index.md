---
title: GitLab CI: artifacts management
url: https://devopstales.github.io/devops/gitlab-ci-artifacts/
date: 2023-01-16
keywords: devops, devops tools, Infrastructure as Code, IAC, gitlab, ci/cd, Continuous Deployments, Continuous Integration, artifacts management, Gitlab package registry
---


In this post I will show you how you can pass artifacts between  in gitlab CI.

<!--more-->

In the [previous post]() we learned the basics of Gitlab CI. I used docker image as an example of artifact in the previous post, but sometimes you need to create other kind of artifacts lik rpm, deb, or a binary, and use this artifacts in the next stage.

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - echo "File 1" >> ./file1.txt
    - echo "File 2" >> ./file2.txt
    - mkdir dir1
    - echo "dir1 File" >> ./dir1/file.txt
    - mkdir dir2
    - echo "dir2 File" >> ./dir2/file.txt
  artifacts:
    paths:
    - file1.txt
    - dir1/

deploy:
  stage: deploy
  script:
    - ls
    - ls dir1
    - cat file1.txt
    - cat dir1/file.txt
```

We defined a build with a `build` and a `deploy` stage and both has a job with tha same name as tha stage.

In `build` we create 4 files:

```bash
├── dir1
│   └── file.txt
├── dir2
│   └── file.txt
├── file1.txt
└── file2.txt
```

We can use the `artifact` property in the `.gitlab-ci.yml`:

```yaml
  artifacts:
    paths:
    - file1.txt
    - dir1/
    expire_in: 1 week
```

Artifacts is used to specify a list of files and directories which should be attached to the build after success. In this example we keep this files for `1 week`

In `deploy`, the following files files - created in `build` - are available:

```bash
├── dir1
│   └── file.txt
├── file1.txt
```

Then we can use the same file in the deploy stage

### Download job artifacts

You can download job artifacts or view the job archive:

* On the Pipelines page, to the right of the pipeline:

![Gitlab artifacts](/img/include/gitlab-artifacts01.webp)

### Upload artifact to Gitlab Generic package registry

Gitlab has its own package registries to store common types of packages for a longer time:

* Composer
* Conan
* Generic
* Maven
* npm
* NuGet
* PyPI
* RubyGems

In the nex example I will use the Generic package registry to store files:

```yaml
image: curlimages/curl:latest

stages:
  - upload
  - download

upload:
  stage: upload
  script:
    - 'curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file file1.txt "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/my_package/0.0.1/file1.txt"'

download:
  stage: download
  script:
    - 'wget --header="JOB-TOKEN: $CI_JOB_TOKEN" ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/my_package/0.0.1/file1.txt'
```
