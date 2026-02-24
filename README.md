# Discover Dollar

A simple CRUD app built with Angular, Node.js, and MongoDB. Everything runs in Docker and deploys automatically via GitHub Actions whenever you push to the `dev` branch.

---

## What's inside

- **Frontend** — Angular 15, built and served via Nginx
- **Backend** — Node.js + Express on port 8080
- **Database** — MongoDB 7 with a persistent volume
- **Reverse Proxy** — Nginx routing traffic to the right service
- **CI/CD** — GitHub Actions builds images, pushes to Docker Hub, and SSHs into the server to redeploy
- **Infrastructure** — AWS EC2 t3.large (2 vCPU, 8GB RAM)

---

## Project structure

```
discover-dollar/
├── backend/
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
├── frontend/
│   ├── Dockerfile
│   └── src/
├── nginx/
│   └── default.conf
├── docker-compose.yml
└── .github/
    └── workflows/
        └── cicd.yml
```

---

## Running locally

Make sure you have Docker and Docker Compose installed. This project was deployed on an AWS **t3.large** EC2 instance — it gives you enough CPU and memory to run all four containers without any tuning.

To run locally:

```bash
git clone https://github.com/<your-username>/discover-dollar.git
cd discover-dollar
docker compose up --build
```

Open `http://localhost` in your browser. That's it.

To stop everything:
```bash
docker compose down
```

To also wipe the MongoDB volume:
```bash
docker compose down -v
```

---

## Docker Compose

This project uses **Docker Compose v2** (v5.0.2). The `version:` field at the top of `docker-compose.yml` is not needed in v2 and will print a warning if left in — it's safe to remove it.

The `docker compose` command (no hyphen) is used instead of the older `docker-compose`.

---

## How the containers fit together

All four services are defined in `docker-compose.yml`. Nginx is the only one exposed to the outside world on port 80 — it proxies `/api/` requests to the backend and everything else to the frontend. The backend talks to MongoDB using Docker's internal DNS (`mongodb:27017`), so there's no hardcoded IP anywhere.

```
Browser → Nginx :80 → Frontend (Angular)
                    → Backend  :8080 → MongoDB :27017
```

---

## Nginx config

Put this in `./nginx/default.conf`:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://frontend:80;
        proxy_set_header Host $host;
    }

    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
    }
}
```

---

## CI/CD

The pipeline lives in `.github/workflows/cicd.yml`. On every push to `dev` it does four things:

1. Logs into Docker Hub
2. Builds and pushes the backend image
3. Builds and pushes the frontend image
4. SSHs into the server and runs `docker compose pull && docker compose up -d`

### Secrets you need to add

Go to **Settings → Secrets and variables → Actions** in your repo and add these:

| Name | What to put |
|------|-------------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username — must be all lowercase |
| `DOCKERHUB_PASSWORD` | Your Docker Hub password or an access token |
| `VM_HOST` | Your server's IP address |
| `VM_USER` | SSH user (usually `ubuntu`) |
| `VM_SSH_KEY` | Your private SSH key |

> One thing worth noting — if the `DOCKERHUB_USERNAME` secret has uppercase letters or a trailing space, the Docker build step will fail with `invalid reference format` even though login succeeds. If that happens, delete the secret and re-type it manually.

---

## Deploying manually

If you ever need to deploy without pushing to GitHub:

```bash
ssh ubuntu@<your-server-ip>
cd /home/ubuntu/discover-dollar
docker compose pull
docker compose up -d
```

---

## Handy commands

```bash
# See what's running
docker ps

# Follow logs for a service
docker compose logs -f backend

# Rebuild just one service
docker compose up --build backend

# Get a shell inside a container
docker exec -it discover-backend sh
```

---

