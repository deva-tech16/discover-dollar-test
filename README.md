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

## Docker files

### backend/Dockerfile

```dockerfile
# Use lightweight Node image
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 8080
CMD ["node", "server.js"]
```

Uses `node:18-alpine` to keep the image small. Only production dependencies are installed — no devDependencies end up in the final image.

---

### frontend/Dockerfile

```dockerfile
# Stage 1 — Build Angular
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build -- --configuration production

# Stage 2 — Serve with Nginx
FROM nginx:alpine
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Two-stage build — the first stage compiles the Angular app, the second stage throws away Node entirely and just serves the compiled output via Nginx. The final image is tiny.

---

### docker-compose.yml

```yaml
services:
  mongodb:
    image: mongo:7
    container_name: mongo
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  backend:
    build: ./backend
    container_name: discover-backend
    restart: always
    environment:
      DB_URL: "mongodb://mongodb:27017/cruddb"
    depends_on:
      - mongodb

  frontend:
    build: ./frontend
    container_name: discover-frontend
    restart: always
    depends_on:
      - backend

  nginx:
    image: nginx:alpine
    container_name: reverse-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend

volumes:
  mongo_data:
```

A few things worth knowing about this setup:

- The `version:` field has been removed — it's deprecated in Docker Compose v2 and just prints a warning if you leave it in
- `depends_on` controls startup order but doesn't wait for a service to be fully ready, just started
- MongoDB data is stored in a named volume (`mongo_data`) so it survives container restarts
- The backend connects to MongoDB using the service name `mongodb` as the hostname — Docker handles the DNS resolution internally
- Only Nginx is exposed to the outside world on port 80. The backend and frontend are only reachable through it

---

## How the containers fit together

```
Browser → Nginx :80 → Frontend (Angular)
                    → Backend  :8080 → MongoDB :27017
```

Nginx is the single entry point. It proxies `/api/` calls to the Node backend and serves everything else from the Angular frontend container.

---

## Nginx config

`./nginx/default.conf`:

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

> If the `DOCKERHUB_USERNAME` secret has uppercase letters or a trailing space, the build step will fail with `invalid reference format` even though login succeeds. If that happens, delete the secret and re-type it manually.

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

## Screenshots

> Add screenshots to a `/screenshots` folder and update these paths.

**CI/CD run**
![cicd](./screenshots/cicd-pipeline.png)

**Docker Hub images**
![dockerhub](./screenshots/dockerhub-images.png)

**App UI**
![ui](./screenshots/app-ui.png)
