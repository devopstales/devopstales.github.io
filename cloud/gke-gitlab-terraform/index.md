---
title: Create K8S cluster with Terraform and GitlabCI
url: https://devopstales.github.io/cloud/gke-gitlab-terraform/
date: 2022-01-30
keywords: terraform, Kubernetes, GCP, Gitlab
---


In this post I will show you how how you can create a K8S cluster with Terraform and GitlabCI.

<!--more-->

### create service account on GCP

```bash
gcloud config set project gke-terraform-test

gcloud iam service-accounts create gitlab-terraform \
--display-name="gitlab-terraform" \
--project=gke-terraform-test

gcloud iam service-accounts add-iam-policy-binding \
gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com \
--member="serviceAccount:gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com" \
--role='roles/compute.networkAdmin'

gcloud iam service-accounts add-iam-policy-binding \
gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com \
--member="serviceAccount:gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com" \
--role='roles/container.admin' \
--project=gke-terraform-test

gcloud iam service-accounts add-iam-policy-binding \
gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com \
--member="serviceAccount:gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com" \
--role='roles/iam.serviceAccountUser' \
--project=gke-terraform-test

gcloud iam service-accounts add-iam-policy-binding \
gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com \
--member="serviceAccount:gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com" \
--role='roles/iam.serviceAccountAdmin' \
--project=gke-terraform-test

gcloud iam service-accounts keys create serviceaccount.json \
--iam-account "gitlab-terraform@gke-terraform-test.iam.gserviceaccount.com"
```

Store the content of the `serviceaccount.json` as base64 encoded value in gitlab variable called `BASE64_GOOGLE_CREDENTIALS`

```bash
base64 serviceaccount.json | tr -d \\n
```


### Create gitlab-ci.yaml

Create CI environment variable: 

`TF_VAR_gitlab_token`:  GitLab personal access token with api scope to add the provisioned cluster to your GitLab group.
`TF_ROOT`: terraform
`TF_VAR_gcp_project`: gke-terraform-test

```yaml
nano gitlab-ci.yaml
include:
  - template: Terraform.gitlab-ci.yml

variables:
  TF_STATE_NAME: production
  TF_CACHE_KEY: production
  TF_ROOT: terraform
  TF_VAR_gcp_project: gke-terraform-test

before_script:
  - export GOOGLE_CREDENTIALS=$(echo $BASE64_GOOGLE_CREDENTIALS | base64 -d)
```

### Create terraform file

```bash
mkdir  terraform
cd
```

Import the `hashicorp/google` provider:

```yaml
nano providers.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "3.79.0"
    }
  }

  required_version = "~> 1.0.3"

  backend "http" {
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

Generate Variables:

```yaml
nano variables.tf
variable "project_id" {
  description = "project id"
}

variable "region" {
  description = "region"
}

variable "gke_num_nodes" {
  default     = 1
  description = "number of gke nodes per zone"
}

variable "machine_type" {
  type        = string
  description = "Type of the node compute engines."
}

variable "disk_size_gb" {
  type        = number
  description = "Size of the node's disk."
}

variable "disk_type" {
  type        = string
  description = "Type of the node's disk."
}

variable "cluster_version" {
  default = "1.20"
}
```

Create network and GKE Cluster:

```yaml
nano main.json
resource "google_compute_network" "vpc" {
  name                    = "gke-test-vpc"
  auto_create_subnetworks = "false"
}

resource "google_compute_subnetwork" "gke-test-network" {
  name          = "gke-test-network"
  region        = var.region
  network       = google_compute_network.vpc.name
  ip_cidr_range = "10.10.10.0/24"
}

resource "google_service_account" "default" {
  account_id   = "service-account-id"
  display_name = "Service Account"
}

resource "google_container_cluster" "primary" {
  name     = "gke-test"
  location = var.region

  min_master_version       = var.cluster_version
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = "gke-test-network"
}


resource "google_container_node_pool" "primary_nodes" {
  name       = "${google_container_cluster.primary.name}-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = var.gke_num_nodes
  version    = var.cluster_version

  management {
    auto_repair  = "true"
    auto_upgrade = "true"
  }

  node_config {
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]

    labels = {
      env = var.project_id
    }

    service_account = google_service_account.default.email
    image_type      = "COS"
    machine_type    = var.machine_type
    disk_size_gb    = var.disk_size_gb
    disk_type       = var.disk_type
    preemptible     = false
    tags         = [
      "gke-node",
    ]

    metadata = {
      disable-legacy-endpoints = "true"
    }
  }
}
```

Add value for the variables:

```yaml
nano terraform.tfvars
project_id   = "gke-terraform-test"
region       = "europe-west1"
gke_num_nodes = 1
machine_type  = "g1-small"
disk_type     = "pd-standard"
disk_size_gb  = 10
```

Configure what data will print in the end of the deploy:

```yaml
nano outputs.tf
output "region" {
  value       = var.region
  description = "GCloud Region"
}

output "project_id" {
  value       = var.project_id
  description = "GCloud Project ID"
}

output "kubernetes_cluster_name" {
  value       = google_container_cluster.primary.name
  description = "GKE Cluster Name"
}
```

If you created all the fles it looks like this:

```bash
cd ..
tree .
.
├── README.md
└── terraform
    ├── providers.tf
    ├── main.tf
    ├── outputs.tf
    ├── terraform.tfvars
    ├── variables.tf

1 directory, 5 files
```

Now you can push to the gitlab project:

```bash
git add -A
git commit -m "base terraform commit"
git push
```


