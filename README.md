## Overview

This project is a **MEAN stack tutorial CRUD application** with a **full DevOps pipeline**:

- **MongoDB** for data storage
- **Express + Node.js** for REST APIs
- **Angular 15** for the frontend
- **Docker + Docker Compose** for containerization
- **Nginx** as a reverse proxy (single entry on port 80)
- **GitHub Actions** CI/CD with:
  - PR validation (build + vulnerability scan, no push, no deploy)
  - Main branch pipeline (build → scan → push → deploy to VM)

The app manages a list of **tutorials** (title, description, published status) with full CRUD and simple search.

 
 `![Architecture](docs/archi.jpeg)`

---

## Architecture & Components

- **Frontend (`frontend/`)**
  - Angular 15 SPA
  - Uses `HttpClient` to call the backend
  - API base URL:
    - Dev: `http://localhost:8080/api/tutorials`
    - Prod: `/api/tutorials` (behind Nginx)

- **Backend (`backend/`)**
  - Node.js + Express
  - REST endpoints under `/api/tutorials`
  - Connects to MongoDB via Mongoose
  - Configurable DB connection via `MONGODB_URI` env var

- **Database**
  - MongoDB container (for Docker / VM)
  - Default DB name: `dd_db`

- **Reverse Proxy (`nginx/nginx.conf`)**
  - Listens on port **80**
  - Routes:
    - `/` → Angular frontend
    - `/api/` → Node.js backend

- **CI/CD**
  - GitHub Actions workflow: `.github/workflows/ci-cd.yml`
  - Uses Docker Buildx, Trivy, and SSH deploy to VM

> **Place for request-flow diagram:**  
> _Insert image here (sequence: Browser → Nginx → Frontend/Backend → MongoDB)._  
> Example: `![Request Flow](docs/request-flow.png)`

---

## Local Development (without Docker)

### Backend – Node.js & Express

```bash
cd backend
npm install
```

Configure MongoDB:

- Default: `mongodb://localhost:27017/dd_db` from `app/config/db.config.js`
- Or override with environment variable:

```bash
export MONGODB_URI="mongodb://localhost:27017/dd_db"
node server.js
```

Backend runs on **http://localhost:8080** by default.

### Frontend – Angular

```bash
cd frontend
npm install
npx ng serve --port 8081
```

Open **http://localhost:8081** and ensure the service is pointing to the backend:

- Dev environment: `src/environments/environment.ts`
  - `apiUrl: 'http://localhost:8080/api/tutorials'`

---

## Docker – Local Development

From the **project root**:

```bash
# Frontend: 80, Backend: 8080, MongoDB: 27017
docker compose up --build
```

- **Frontend:** `http://localhost:80`  
- **Backend API:** `http://localhost:8080`  

Use this mode to validate containers and app behavior locally before moving to Nginx-only port 80.

> **Place for screenshot:**  
> _Insert screenshot of the Angular UI running locally._  
> Example: `![UI Local](docs/ui-local.png)`

---

## Docker – Single Port 80 (Nginx Reverse Proxy)

For a production-like setup with **only port 80 exposed**:

```bash
docker compose -f docker-compose.prod.yml up --build -d
```

- **UI + API:** `http://localhost:80`
- Nginx:
  - `/` → frontend
  - `/api/` → backend

Angular **production build** uses:

- `src/environments/environment.prod.ts` → `apiUrl: '/api/tutorials'`

so all API calls go through Nginx (`/api/...`).

---

## Docker Hub – Build & Push Images Manually

1. Log in:

```bash
docker login
```

2. Build and tag (replace `YOUR_DOCKERHUB_USERNAME`):

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/dd-backend:latest ./backend
docker build -t YOUR_DOCKERHUB_USERNAME/dd-frontend:latest ./frontend
docker push YOUR_DOCKERHUB_USERNAME/dd-backend:latest
docker push YOUR_DOCKERHUB_USERNAME/dd-frontend:latest
```

These same image names are used by the **VM deployment** compose file.

---

## Deploy on Ubuntu/DigitalOcean VM (Port 80 via Nginx)

### 1. Prepare VM

- Provision a VM (e.g. DigitalOcean droplet, AWS EC2)
- Open:
  - **Port 80** (HTTP)
  - **Port 22** (SSH)

Install Docker & Docker Compose:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
# Log out & log back in so docker works without sudo
```

### 2. Clone the Repository

On the VM:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO.git
cd YOUR_REPO
```

This gives you:

- `docker-compose.hub.yml`
- `nginx/nginx.conf`

### 3. Run using Docker Hub images

```bash
export DOCKERHUB_USERNAME=YOUR_DOCKERHUB_USERNAME
docker compose -f docker-compose.hub.yml pull
docker compose -f docker-compose.hub.yml up -d
```

- Open `http://VM_PUBLIC_IP` → app and API via Nginx on port 80
- MongoDB runs as a **container**; no host install needed.

> **Place for infra diagram:**  
> _Insert image here (cloud VM with containers: nginx, frontend, backend, mongo)._  
> Example: `![VM Deployment](docs/vm-deployment.png)`

---

## CI/CD – GitHub Actions

Workflow file: `.github/workflows/ci-cd.yml`

### Pull Requests (`pull_request` → main/master)

- **Build (multi-stage)** backend & frontend with Docker Buildx
- **Use cache** (`cache-from/cache-to`) for faster builds
- **Trivy scan** (CRITICAL/HIGH) on:
  - `backend-pr:${{ github.sha }}`
  - `frontend-pr:${{ github.sha }}`
- **No push, no deploy**  
  PR must pass builds + security scan before merge.

### Main/Master (`push` → main/master)

1. **Build images (no push yet)**:
   - Backend → `backend-ci:${SHA}`
   - Frontend → `frontend-ci:${SHA}`
2. **Trivy scan (block on CRITICAL vulns)**:
   - Scans `backend-ci:${SHA}` and `frontend-ci:${SHA}`
3. **If scans pass → push to Docker Hub**:
   - Backend:
     - `${DOCKERHUB_USERNAME}/dd-backend:${SHA}`
     - `${DOCKERHUB_USERNAME}/dd-backend:latest`
   - Frontend:
     - `${DOCKERHUB_USERNAME}/dd-frontend:${SHA}`
     - `${DOCKERHUB_USERNAME}/dd-frontend:latest`

### Automatic Deploy (`deploy-vm` job)

After images are pushed successfully on a **push to `main`/`master`**, the `deploy-vm` job:

- SSHs into the VM using `VM_HOST`, `VM_USER`, `VM_SSH_KEY`
- Runs `git pull` inside `VM_APP_PATH`
- Executes:
  - `docker compose -f docker-compose.hub.yml pull`
  - `docker image prune -af --filter "until=24h" || true`
  - `docker compose -f docker-compose.hub.yml up -d`

This keeps the VM in sync with the latest `main` images. To disable auto‑deploy, comment out the `deploy-vm` job in `.github/workflows/ci-cd.yml`.

### Required Secrets / Variables

In GitHub repo → **Settings → Secrets and variables → Actions**:

| Name               | Type      | Description |
|--------------------|-----------|-------------|
| `DOCKERHUB_USERNAME` | Variable | Your Docker Hub username (used in tags) |
| `DOCKERHUB_TOKEN`   | Secret   | Docker Hub token/password for `docker login` |
| `VM_HOST`           |  Variable | VM IP/hostname (for deploy job) |
| `VM_USER`           | Secret   | SSH user (e.g. `root` on DigitalOcean) |
| `VM_SSH_KEY`        | Secret   | Private SSH key that can access the VM |
| `VM_APP_PATH`       | Secret   | Path on VM where repo is cloned (e.g. `/root/crud-dd-task-mean-app`) |

If you **only want build + scan + push**, keep the deploy job commented out (as currently).

---

## File & Directory Overview

| Path / File                        | Description |
|------------------------------------|-------------|
| `backend/`                         | Node.js/Express API |
| `backend/server.js`                | Express app entry point |
| `backend/app/config/db.config.js`  | MongoDB URL (uses `MONGODB_URI` if set) |
| `backend/Dockerfile`               | Multi-stage Dockerfile for backend |
| `frontend/`                        | Angular 15 app |
| `frontend/Dockerfile`              | Multi-stage Dockerfile (build + Nginx) |
| `frontend/src/environments/*`      | Dev/prod API URLs for Angular |
| `nginx/nginx.conf`                 | Reverse proxy: `/` → frontend, `/api/` → backend |
| `docker-compose.yml`               | Local dev: frontend:80, backend:8080, mongo |
| `docker-compose.prod.yml`          | Nginx on 80, frontend+backend behind it (build locally) |
| `docker-compose.hub.yml`           | Use images from Docker Hub on VM |
| `.github/workflows/ci-cd.yml`      | CI/CD workflow (PR + main) |

> **Place for final result screenshot:**  
> _Insert image here (UI after creating a few tutorials)._  
> Example: `![App Screenshot](docs/app-screenshot.png)`

