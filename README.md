In this DevOps task, you need to build and deploy a full-stack CRUD application using the MEAN stack (MongoDB, Express, Angular 15, and Node.js). The backend will be developed with Node.js and Express to provide REST APIs, connecting to a MongoDB database. The frontend will be an Angular application utilizing HTTPClient for communication.

The application will manage a collection of tutorials, where each tutorial includes an ID, title, description, and published status. Users will be able to create, retrieve, update, and delete tutorials. Additionally, a search box will allow users to find tutorials by title.

---

## Project setup (local without Docker)

### Node.js Server

```bash
cd backend
npm install
```

You can update the MongoDB credentials by modifying the `app/config/db.config.js` file (or set `MONGODB_URI` env var).

```bash
node server.js
```

### Angular Client

```bash
cd frontend
npm install
npx ng serve --port 8081
```

You can modify `src/app/services/tutorial.service.ts` or `src/environments/environment.ts` to adjust how the frontend talks to the backend.

Navigate to `http://localhost:8081`

---

## Docker – run locally

From the project root:

```bash
# Build and run (frontend on port 80, backend on 8080, MongoDB on 27017)
docker compose up --build -d
```

- Frontend: http://localhost:80  
- Backend API: http://localhost:8080  

---

## Docker – single port 80 (Nginx reverse proxy)

To serve the whole app on port 80 (e.g. for production or VM):

```bash
docker compose -f docker-compose.prod.yml up --build -d
```

- App (UI + API): http://localhost:80  
- `/api/` is proxied to the backend.

---

## Docker Hub – build and push images

1. Log in: `docker login`
2. Build and tag (replace `YOUR_DOCKERHUB_USERNAME` with your Docker Hub username):

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/dd-backend:latest ./backend
docker build -t YOUR_DOCKERHUB_USERNAME/dd-frontend:latest ./frontend
docker push YOUR_DOCKERHUB_USERNAME/dd-backend:latest
docker push YOUR_DOCKERHUB_USERNAME/dd-frontend:latest
```

---

## Deploy on Ubuntu VM (port 80, Nginx)

1. **Provision** an Ubuntu VM (AWS, Azure, etc.) and open port 80 (and 22 for SSH).

2. **On the VM**, install Docker and Docker Compose:

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
# Log out and back in so docker runs without sudo
```

3. **Clone the repo** on the VM (so you have `nginx/nginx.conf` and `docker-compose.hub.yml`):

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
```

4. **Set your Docker Hub username** and run using pre-built images:

```bash
export DOCKERHUB_USERNAME=YOUR_DOCKERHUB_USERNAME
docker compose -f docker-compose.hub.yml pull
docker compose -f docker-compose.hub.yml up -d
```

5. Open **http://VM_PUBLIC_IP** in a browser. The app and API are served on port 80 via Nginx.

**Database:** MongoDB runs in Docker (Mongo image in the same compose file). No need to install MongoDB on the host.

---

## CI/CD (GitHub Actions)

The workflow in `.github/workflows/ci-cd.yml`:

- **On push to `main` or `master`:** builds backend and frontend Docker images and pushes them to Docker Hub.
- **Deploy job:** SSHs into your VM, pulls the repo, runs `docker compose -f docker-compose.hub.yml pull` and `up -d` so the VM runs the latest images.

### GitHub repository secrets

In the repo: **Settings → Secrets and variables → Actions**, add:

| Secret              | Description                                      |
|---------------------|--------------------------------------------------|
| `DOCKERHUB_USERNAME`| Your Docker Hub username                         |
| `DOCKERHUB_TOKEN`   | Docker Hub access token (Account → Security)     |
| `VM_HOST`           | VM public IP or hostname                         |
| `VM_USER`           | SSH user (e.g. `ubuntu`)                         |
| `VM_SSH_KEY`        | Full private key for SSH (e.g. contents of `.pem`) |
| `VM_APP_PATH`       | Path on VM where repo is cloned (e.g. `/home/ubuntu/crud-dd-task-mean-app`) |

If you only want build-and-push (no deploy), you can remove or comment out the `deploy-vm` job in the workflow; the build-and-push job will still run.

---

## File overview

| File / folder              | Purpose |
|----------------------------|--------|
| `backend/Dockerfile`       | Multi-stage Dockerfile for Node API |
| `frontend/Dockerfile`      | Multi-stage Dockerfile for Angular (build + nginx) |
| `docker-compose.yml`       | Local run: frontend:80, backend:8080, mongo |
| `docker-compose.prod.yml`  | Nginx on 80, frontend + backend behind it |
| `docker-compose.hub.yml`   | Same as prod but uses Docker Hub images (for VM) |
| `nginx/nginx.conf`         | Nginx reverse proxy: `/` → frontend, `/api/` → backend |
| `.github/workflows/ci-cd.yml` | Build, push to Docker Hub, optional deploy to VM |

ci-cd.yml have 2 type of the triggers