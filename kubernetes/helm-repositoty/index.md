---
title: Create a Helm reposirory with GitHub Pages
url: https://devopstales.github.io/kubernetes/helm-repositoty/
date: 2021-07-25
keywords: kubernetes, helm, repository, GitHub Repository, GitHub Pages
---


In this post I will show you how you can host your own Helm repository with GitHub Pages.
<!--more-->

## Create a new GitHub Repository

Log into GitHub and create a [new repository](https://github.com/new) called helm-charts. I chose to hav a README file and an Apache2 licence in mye repository.

Clone the repository to start working.

```bash
git clone git@github.com:devopstales/helm-charts.git
cd helm-charts

tree
.
├── LICENSE
└── README.md
```

Create a hem chart in the repository:

```bash
mkdir charts
helm create charts/chart1
helm create charts/chart2

tree
.
├── charts
│   ├── chart1
│   │   ├── charts
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── deployment.yaml
│   │   │   ├── _helpers.tpl
│   │   │   ├── ingress.yaml
│   │   │   ├── NOTES.txt
│   │   │   ├── service.yaml
│   │   │   └── tests
│   │   │       └── test-connection.yaml
│   │   └── values.yaml
│   └── chart2
│       ├── charts
│       ├── Chart.yaml
│       ├── templates
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   ├── ingress.yaml
│       │   ├── NOTES.txt
│       │   ├── service.yaml
│       │   └── tests
│       │       └── test-connection.yaml
│       └── values.yaml
├── LICENSE
└── README.md
```

### Push to GitHub:

```bash
echo ".deploy" >> .gitignore
git add . --all
git commit -m 'Initial Commit'
git push origin main
```

Create brach for GitHub Pages and release:

```bash
git checkout --orphan gh-pages
Switched to a new branch 'gh-pages'

rm -rf charts
git add . --all
git commit -m 'initial gh-pages'
git push origin gh-pages
git checkout main
```

Next enable GitHub Pages i the repository settings. After a few minutes you should have a default rendering on your README.md at the provided URL.

## Use chart-releaser

Yo can create a chart Helm repository by usin the `helm package` and `helm repo` commands but you can simplify your life by using `chart-releaser`.

### Install for Lnux:

```bash
cd /tmp
curl -sSL https://github.com/helm/chart-releaser/releases/download/v1.2.1/chart-releaser_1.2.1_linux_amd64.tar.gz | tar xzf -
mv cr ~/bin/cr
cr help
```

## Install for Mac osX:

```bash
$ brew tap helm/tap
$ brew install chart-releaser
```

### Usage:

The `cr index` will create the appropriate `index.yaml` and `cr upload` will upload the packages to GitHub Releases. Fot theat you need a GitHub Token.
In your browser go to your [github developer settings](https://github.com/settings/tokens) and create a new personal access token and add full access to the repo.

Create an environment variable for the token:

```bash
export CH_TOKEN=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sdf

helm package charts/{chart1,chart2} --destination .deploy

cr upload -o devopstales -r helm-charts -p .deploy
git checkout gh-pages
cr index -i ./index.yaml -p .deploy -o devopstales -r helm-charts -c https://devopstales.github.io/helm-charts/

git add index.yaml
git commit -m 'release 0.1.0'
git push origin gh-pages
```

### Update the README.md with instructions to usage

```bash
nano README.md
git add README.md
git commit -m 'update readme with instructions'
git push origin gh-pages
```

