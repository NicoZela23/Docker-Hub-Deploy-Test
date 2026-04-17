# FlavorFlex — DevOps Showcase

## Overview

FlavorFlex is a React recipe app used as a vehicle to demonstrate a complete DevOps workflow: containerization with Docker, automated CI/CD via GitHub Actions, and image distribution through Docker Hub.

The app itself is intentionally simple — the focus is the infrastructure and deployment pipeline around it.

---

## Table of Contents

- [DevOps Architecture](#devops-architecture)
- [CI/CD Pipeline](#cicd-pipeline)
- [Docker Setup](#docker-setup)
- [Running Locally (No Docker)](#running-locally-no-docker)
- [Run with Docker (Local Build)](#run-with-docker-local-build)
- [Pull from Docker Hub](#pull-from-docker-hub)

---

## DevOps Architecture

```
Developer pushes to master
        │
        ▼
┌─────────────────────────────────────────────┐
│           GitHub Actions CI/CD              │
│                                             │
│  1. Checkout code                           │
│  2. Setup Node.js 20.15.0                   │
│  3. Cache node_modules (package-lock.json)  │
│  4. Install dependencies (npm ci)           │
│  5. Run Jest unit tests                     │
│  6. Run ESLint static analysis              │
│  7. Build production bundle (Vite)          │
│  8. Build Docker image (multi-stage)        │
│  9. Push to Docker Hub (tagged by SHA)      │
└─────────────────────────────────────────────┘
        │
        ▼
  Docker Hub Registry
  nicozela23/final-project-devops-nzo:<git-sha>
```

---

## CI/CD Pipeline

Pipeline defined in `.github/workflows/docker-publish.yml`.

**Triggers:** push or pull request to `master`.

**Key decisions:**

| Step | Tool | Why |
|------|------|-----|
| Dependency install | `npm ci` | Reproducible install, respects lock file |
| Caching | `actions/cache@v3` | Caches `node_modules` by `package-lock.json` hash — skips install on unchanged deps |
| Tests | Jest `--ci --runInBand` | Serial execution, no interactive prompts, exits non-zero on failure |
| Linting | ESLint | `continue-on-error: true` — warns without blocking deploy |
| Image tag | `github.sha` | Every push produces a unique, traceable image |
| Auth | GitHub Secrets | `DOCKER_USERNAME` / `DOCKER_PASSWORD` — credentials never hardcoded |

**Secrets required in GitHub repo settings:**

```
DOCKER_USERNAME   → Docker Hub username
DOCKER_PASSWORD   → Docker Hub password or access token
```

---

## Docker Setup

### Multi-Stage Dockerfile

The `Dockerfile` uses a two-stage build to keep the final image small:

```dockerfile
# Stage 1 — Build
FROM node:20.15.0-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . ./
RUN npm run build          # Vite outputs to /app/dist

# Stage 2 — Serve
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Why multi-stage?**
- Stage 1 (Node) compiles the app — this image is ~400MB+ with all dev dependencies.
- Stage 2 (nginx:alpine) only copies the compiled static files — final image is ~25MB.
- No Node.js, no source code, no dev dependencies in production.

**Port mapping:** container exposes `80`, mapped to `8080` on the host.

---

## Running Locally (No Docker)

**Prerequisites:** Node.js 20.15.0, Git

```bash
git clone https://github.com/NicoZela23/final-project-devops-nzo.git
cd final-project-devops-nzo
npm install
npm run dev
```

---

## Run with Docker (Local Build)

**Prerequisites:** Docker

```bash
# Clone
git clone https://github.com/NicoZela23/final-project-devops-nzo.git
cd final-project-devops-nzo

# Build image locally
docker build -t final-project-devops-nzo:local .

# Verify image was created
docker images | grep final-project-devops-nzo

# Run container (host port 8080 → container port 80)
docker run -d -p 8080:80 final-project-devops-nzo:local

# Verify container is running
docker ps
```

Open [http://localhost:8080](http://localhost:8080)

**Stop container:**
```bash
docker stop $(docker ps -q --filter ancestor=final-project-devops-nzo:local)
```

---

## Pull from Docker Hub

Images are pushed automatically by the CI/CD pipeline on every merge to `master`, tagged with the Git commit SHA.

```bash
# Login (first time)
docker login

# Pull latest image
docker pull nicozela23/final-project-devops-nzo

# Run it
docker run -d -p 8080:80 nicozela23/final-project-devops-nzo

# Confirm running
docker ps
```

Open [http://localhost:8080](http://localhost:8080)

**Docker Hub repo:** [hub.docker.com/r/nicozela23/final-project-devops-nzo](https://hub.docker.com/r/nicozela23/final-project-devops-nzo)

---

## Tech Stack

| Layer | Tool |
|-------|------|
| Frontend | React 18, TypeScript, Vite |
| Styling | Tailwind CSS |
| Testing | Jest, React Testing Library |
| Linting | ESLint |
| Containerization | Docker (multi-stage, nginx:alpine) |
| Registry | Docker Hub |
| CI/CD | GitHub Actions |
