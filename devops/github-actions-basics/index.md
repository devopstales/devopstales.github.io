---
title: GitHub Actions: Basics
url: https://devopstales.github.io/devops/github-actions-basics/
date: 2026-03-07
keywords: devops, devops tools, GitHub, ci/cd, Continuous Deployments, Continuous Integration, artifacts management, GitHub Actions, GitHub pipelines
---


In this post I will show you how you can use GitHub Actions for CI/CD and pass artifacts between jobs.

<!--more-->

### What is GitHub Actions?

GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline with GitHub. 

You can configure a GitHub Actions **workflow** to be triggered when an **event** occurs in your repository, such as a pull request being opened or an issue being created. Your workflow contains one or more **jobs** which can run in sequential order or in parallel. Each job will run inside its own virtual machine **runner**, or inside a container, and has one or more **steps** that either run a script that you define or run an **action**, which is a reusable extension that can simplify your workflow. You can find the available action in the [GitHub marketplace](https://github.com/marketplace?type=actions)

### Create an Example Workflow

GitHub Actions uses YAML syntax to define the workflow. Each workflow is stored as a separate YAML file in your code repository, in a directory named `.github/workflows`.

```yaml
mkdir .github/workflows
nano .github/workflows/cicd.yaml
---
name: learn-github-actions

# Trigger
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main ]

jobs:
  # Job definition. You can create multiple jobs
  build:
    name: 'Build'
    runs-on: ubuntu-24.04
    steps:
      - name: "Checkout code"
        uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm install -g bats
      - run: bats -v
      # Share between jobs
      - uses: actions/upload-artifact@v4
        with:
          name: build-data
          path: dist/
          retention-days: 5

  deploy-dev:
    name: "Deploy To Dev"
    # Add job called build as dependency
    needs: build
    runs-on: ubuntu-24.04
    environment: development
    steps:
    # Download from previous job
      - uses: actions/download-artifact@v4
        with:
          name: build-data
          path: dist/
      - name: Deploy
        run: echo "Deploying to development environment"
```

If you created the file, you need to commit and push to the GitHub repository. Then it will run automatically each time someone pushes a change to the repository. When your **workflow** is **triggered**, a **workflow run** is created that executes the workflow. After a workflow run has started, you can see a visualization graph of the run's progress and view each step's activity on GitHub. To find this click **Actions** in the repository.

![GitHub Actions](/img/include/github-actions-basics01.webp)

![GitHub Actions](/img/include/github-actions-basics02.webp)

![GitHub Actions](/img/include/github-actions-basics03.webp)

![GitHub Actions](/img/include/github-actions-basics04.webp)

![GitHub Actions](/img/include/github-actions-basics05.webp)

---

* https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions
* https://github.com/actions
