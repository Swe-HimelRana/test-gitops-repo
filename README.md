# CI/CD + GitOps Practice Project

A simple hands-on project for learning **GitHub Actions, Docker, GitHub Container Registry (GHCR), Helm, Argo CD, and Kubernetes/k3s** using a GitOps workflow.

## Architecture

```text
Developer
   │
   │ git push
   ▼
GitHub Application Repository
   │
   │ GitHub Actions
   │
   ├── Build Docker image
   │
   ├── Push image to GHCR
   │
   └── Update Helm repository
          │
          │ git commit + push
          ▼
GitOps / Helm Repository
   │
   │ Argo CD watches repository
   ▼
Argo CD
   │
   │ sync
   ▼
k3s Kubernetes Cluster
   │
   ▼
Landing Page
```

---

## Repositories

This project uses two separate Git repositories.

### 1. Application Repository

This repository contains the application source code and CI workflow.

```text
test-gitops-repo/
├── .github/
│   └── workflows/
│       └── docker.yml
├── Dockerfile
├── index.html
└── README.md
```

The application repository is responsible for:

* HTML source code
* Docker image creation
* Docker image publishing
* Updating the GitOps repository

### 2. GitOps / Helm Repository

The Kubernetes configuration is maintained separately.

```text
test-gitops-helm-repo/
└── helm/
    ├── Chart.yaml
    ├── values.yml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
```

The GitOps repository represents the **desired state** of the Kubernetes application.

Argo CD continuously monitors this repository.

---

# Application

The application is a simple static HTML landing page.

The page is served using Nginx inside a Docker container.

## Dockerfile

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

The Docker image is based on the lightweight `nginx:alpine` image.

The HTML file is copied into Nginx's default web root:

```text
/usr/share/nginx/html/index.html
```

---

# Local Docker Test

Build the image:

```bash
docker build -t landing-page:test .
```

Run it:

```bash
docker run --rm -p 8080:80 landing-page:test
```

Open:

```text
http://localhost:8080
```

---

# CI — GitHub Actions

The CI pipeline is located at:

```text
.github/workflows/docker.yml
```

The workflow runs whenever code is pushed to a branch.

```yaml
on:
  push:
    branches:
      - '**'
```

Therefore, pushing to:

```text
main
development
feature/new-header
```

can trigger the workflow.

---

# Docker Image Naming

Docker images are pushed to GitHub Container Registry.

The image repository is:

```text
ghcr.io/swe-himelrana/landing-page
```

The workflow generates two tags.

## Branch tag

For the `main` branch:

```text
ghcr.io/swe-himelrana/landing-page:main
```

For example:

```text
development
```

becomes:

```text
ghcr.io/swe-himelrana/landing-page:development
```

Branch names are sanitized so they can safely be used as Docker tags.

---

# SHA Image Tag

The workflow also creates a tag using the Git commit SHA.

Example:

```text
main-a82f31c
```

The resulting image is:

```text
ghcr.io/swe-himelrana/landing-page:main-a82f31c
```

The SHA-based tag is important because it identifies the exact source revision used to build the image.

The branch tag:

```text
main
```

is mutable.

The SHA tag:

```text
main-a82f31c
```

points to a specific build.

---

# CI Pipeline

The GitHub Actions workflow performs the following steps.

## 1. Checkout application source

```text
GitHub
   ↓
actions/checkout
   ↓
Application source
```

## 2. Login to GHCR

GitHub Actions authenticates with GitHub Container Registry using:

```text
GITHUB_TOKEN
```

## 3. Generate image tags

The workflow obtains:

```text
GITHUB_REF_NAME
GITHUB_SHA
```

and creates Docker-compatible tags.

Example:

```text
Branch: main
SHA: a82f31c...
```

Result:

```text
main
main-a82f31c
```

## 4. Build Docker image

```bash
docker build .
```

## 5. Push Docker image

Both tags are pushed to GHCR:

```text
ghcr.io/swe-himelrana/landing-page:main
ghcr.io/swe-himelrana/landing-page:main-a82f31c
```

---

# GitOps Repository Update

After successfully pushing the Docker image, the workflow checks out the separate GitOps repository.

Repository:

```text
Swe-HimelRana/test-gitops-helm-repo
```

The Helm values file contains the image configuration:

```yaml
image:
  repository: ghcr.io/swe-himelrana/landing-page
  tag: main
```

The CI workflow automatically replaces the tag with the newly created SHA tag.

For example:

```yaml
image:
  repository: ghcr.io/swe-himelrana/landing-page
  tag: main-a82f31c
```

The workflow then:

1. Changes `helm/values.yml`
2. Creates a Git commit
3. Pushes the commit to the GitOps repository

Example commit:

```text
Update landing-page image to main-a82f31c
```

---

# Why Update Git Instead of Deploying Directly?

This project intentionally does **not** have GitHub Actions run:

```bash
kubectl apply
```

Instead, GitHub Actions changes the desired state in Git.

The GitOps repository becomes the source of truth.

```text
Git repository
      │
      │ desired state
      ▼
   Argo CD
      │
      │ reconciliation
      ▼
 Kubernetes
```

This is one of the main concepts being practiced in this project.

---

# CD — Argo CD

Argo CD will run inside the k3s cluster.

Its job is to continuously monitor the GitOps repository.

For example:

```text
GitOps repository

image:
  repository: ghcr.io/swe-himelrana/landing-page
  tag: main-a82f31c
```

Argo CD detects the Git change and compares it with the currently deployed state.

If Kubernetes is running:

```text
main-1234567
```

while Git specifies:

```text
main-a82f31c
```

Argo CD detects that the cluster is out of sync.

It then synchronizes the cluster to the Git-defined state.

---

# Complete Deployment Flow

A normal application update will work like this:

```text
1. Developer changes index.html
             │
             ▼
2. git commit
             │
             ▼
3. git push
             │
             ▼
4. GitHub Actions starts
             │
             ▼
5. Docker image is built
             │
             ▼
6. Image pushed to GHCR
             │
             ├── :main
             │
             └── :main-a82f31c
             │
             ▼
7. GitHub Actions updates Helm values.yml
             │
             ▼
8. GitOps repository receives commit
             │
             ▼
9. Argo CD detects Git change
             │
             ▼
10. Argo CD synchronizes Helm chart
             │
             ▼
11. Kubernetes Deployment changes
             │
             ▼
12. New Pod starts
             │
             ▼
13. New landing page is live
```

---

# CI vs CD

This project separates CI and CD responsibilities.

## Continuous Integration

GitHub Actions handles:

```text
Source Code
     ↓
Build
     ↓
Docker Image
     ↓
GHCR
     ↓
Update GitOps Repository
```

## Continuous Deployment

Argo CD handles:

```text
GitOps Repository
       ↓
Argo CD
       ↓
Kubernetes
       ↓
Running Application
```

This separation is intentional.

---

# GitOps Repository as Source of Truth

The GitOps repository should always describe what should be running in Kubernetes.

For example:

```yaml
image:
  repository: ghcr.io/swe-himelrana/landing-page
  tag: main-a82f31c
```

means:

> Kubernetes should run this exact application image.

If the image tag changes in Git:

```text
main-a82f31c
        ↓
main-b93d821
```

Argo CD detects the difference and reconciles the cluster.

---

# Authentication

The application repository needs permission to push Docker images to GHCR.

GitHub Actions uses:

```text
GITHUB_TOKEN
```

with package write permission.

The workflow contains:

```yaml
permissions:
  contents: read
  packages: write
```

To modify the separate GitOps repository, a separate credential is used.

For this practice project:

```text
GITOPS_TOKEN
```

is stored as a GitHub Actions repository secret.

The token should have access only to the GitOps repository and should have:

```text
Contents: Read and write
```

Never commit the token directly into the repository.

---

# Kubernetes / k3s

The final deployment target is a k3s Kubernetes cluster.

The Helm chart will eventually create:

```text
Deployment
Service
```

The Deployment runs the landing page container.

The Service provides network access to the Pods.

The final architecture will look like:

```text
                    Internet
                       │
                       ▼
                  Kubernetes
                       │
                 ┌─────┴─────┐
                 │   Service │
                 └─────┬─────┘
                       │
              ┌────────┴────────┐
              │                 │
           Pod 1             Pod 2
              │                 │
              └────────┬────────┘
                       │
                 Nginx container
                       │
                    index.html
```

---

# Helm

Helm is used to package the Kubernetes manifests.

The chart will contain:

```text
helm/
├── Chart.yaml
├── values.yml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

`values.yml` contains configurable values such as:

```yaml
image:
  repository: ghcr.io/swe-himelrana/landing-page
  tag: main-a82f31c

replicaCount: 1
```

The Kubernetes templates consume these values.

This allows the CI pipeline to update the image without modifying the Kubernetes Deployment template itself.

---

# Learning Objectives

This project is intended to practice:

* Git
* GitHub
* GitHub Actions
* CI pipelines
* Docker
* Docker image tagging
* GitHub Container Registry
* Git commit SHA versioning
* Helm
* Kubernetes
* k3s
* GitOps
* Argo CD
* Continuous Deployment
* Kubernetes reconciliation
* Immutable application versions

---

# Planned Project Stages

## Stage 1 — Application

* [x] Create HTML landing page
* [x] Create Dockerfile
* [x] Test Docker image locally

## Stage 2 — CI

* [x] Create GitHub Actions workflow
* [x] Build Docker image
* [x] Generate branch tag
* [x] Generate SHA-based tag
* [x] Push image to GHCR

## Stage 3 — GitOps

* [x] Create separate GitOps repository
* [x] Create Helm chart
* [x] Automatically update `helm/values.yml`
* [x] Commit updated image tag
* [x] Push changes to GitOps repository

## Stage 4 — Kubernetes

* [x] Create Deployment template
* [x] Create Service template
* [x] Test Helm chart locally
* [x] Install chart manually into k3s

## Stage 5 — Argo CD

* [x] Install Argo CD in k3s
* [x] Configure Argo CD Application
* [x] Connect Argo CD to GitOps repository
* [x] Deploy Helm chart through Argo CD
* [x] Verify synchronization

## Stage 6 — Full Automation

The final goal is:

```text
git push
   ↓
GitHub Actions
   ↓
Docker build
   ↓
GHCR
   ↓
Update Helm values
   ↓
GitOps commit
   ↓
Argo CD
   ↓
k3s
   ↓
New application version
```

At the end, changing only `index.html` and pushing to GitHub should be enough to automatically deploy the new version to Kubernetes.

---

# Project Philosophy

This project intentionally uses a simple application so that the focus remains on the infrastructure and deployment pipeline.

The application itself is not the interesting part.

The important part is learning how these systems work together:

```text
Git
 │
 ├── Application source
 │
 └── Desired Kubernetes state
          │
          ▼
     GitHub Actions
          │
          ▼
        Docker
          │
          ▼
         GHCR
          │
          ▼
        Helm
          │
          ▼
       Argo CD
          │
          ▼
       Kubernetes
          │
          ▼
         k3s
```

This provides a foundation for applying the same architecture to larger applications and production workloads.
