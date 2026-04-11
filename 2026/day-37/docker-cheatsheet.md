# Docker Cheat Sheet

## Container Commands

| Command | What it does |
|---|---|
| `docker run -it nginx bash` | Run container interactively with a shell |
| `docker run -d -p 8080:80 nginx` | Run container detached, map host:container port |
| `docker run --name myapp -d nginx` | Run with a custom name |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker stop <name/id>` | Gracefully stop a running container |
| `docker kill <name/id>` | Force-stop a container immediately |
| `docker rm <name/id>` | Remove a stopped container |
| `docker rm -f <name/id>` | Force-remove a running container |
| `docker exec -it <name> bash` | Open shell inside running container |
| `docker logs <name>` | View container logs |
| `docker logs -f <name>` | Follow (tail) container logs |
| `docker inspect <name>` | Full JSON details about a container |
| `docker stats` | Live resource usage for all containers |

---

## Image Commands

| Command | What it does |
|---|---|
| `docker pull nginx:latest` | Pull image from Docker Hub |
| `docker build -t myapp:1.0 .` | Build image from Dockerfile in current dir |
| `docker build -t myapp:1.0 -f Dockerfile.prod .` | Build using a specific Dockerfile |
| `docker images` or `docker image ls` | List local images |
| `docker rmi <image>` | Remove an image |
| `docker tag myapp:1.0 user/myapp:1.0` | Tag image for a registry |
| `docker push user/myapp:1.0` | Push image to Docker Hub |
| `docker history <image>` | Show image layers |
| `docker image inspect <image>` | Full details about an image |

---

## Volume Commands

| Command | What it does |
|---|---|
| `docker volume create mydata` | Create a named volume |
| `docker volume ls` | List all volumes |
| `docker volume inspect mydata` | Details about a volume |
| `docker volume rm mydata` | Remove a volume |
| `docker run -v mydata:/app/data nginx` | Mount named volume into container |
| `docker run -v $(pwd):/app nginx` | Bind mount current dir into container |

---

## Network Commands

| Command | What it does |
|---|---|
| `docker network create mynet` | Create a custom bridge network |
| `docker network ls` | List all networks |
| `docker network inspect mynet` | Details about a network |
| `docker network connect mynet <container>` | Connect running container to a network |
| `docker network disconnect mynet <container>` | Disconnect container from a network |
| `docker run --network mynet nginx` | Run container on a specific network |

---

## Compose Commands

| Command | What it does |
|---|---|
| `docker compose up` | Start all services (foreground) |
| `docker compose up -d` | Start all services (detached) |
| `docker compose up --build` | Rebuild images then start |
| `docker compose down` | Stop and remove containers + networks |
| `docker compose down -v` | Same as above + removes named volumes |
| `docker compose ps` | List compose services and their status |
| `docker compose logs -f` | Follow logs for all services |
| `docker compose logs -f web` | Follow logs for a specific service |
| `docker compose build` | Build/rebuild images without starting |
| `docker compose exec web bash` | Shell into a running compose service |
| `docker compose restart web` | Restart a specific service |

---

## Cleanup Commands

| Command | What it does |
|---|---|
| `docker system df` | Show disk usage (images, containers, volumes) |
| `docker system prune` | Remove all stopped containers, unused networks, dangling images |
| `docker system prune -a` | Same + removes all unused images (not just dangling) |
| `docker container prune` | Remove all stopped containers |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove all unused images |
| `docker volume prune` | Remove all unused volumes |
| `docker network prune` | Remove all unused networks |

---

## Dockerfile Instructions

| Instruction | What it does |
|---|---|
| `FROM node:20-alpine` | Base image — always the first instruction |
| `WORKDIR /app` | Set working directory for subsequent instructions |
| `COPY package*.json ./` | Copy files from host into image |
| `ADD archive.tar.gz /app` | Like COPY but also auto-extracts archives and supports URLs |
| `RUN npm install` | Execute a command during build; creates a new layer |
| `ENV NODE_ENV=production` | Set environment variable baked into the image |
| `ARG BUILD_VERSION` | Build-time variable (not available at runtime) |
| `EXPOSE 3000` | Document which port the container listens on |
| `VOLUME ["/data"]` | Declare a mount point for external volumes |
| `CMD ["node", "server.js"]` | Default command — can be overridden at `docker run` |
| `ENTRYPOINT ["node"]` | Fixed executable — CMD or run args become its arguments |
| `HEALTHCHECK CMD curl -f http://localhost/ || exit 1` | Define how Docker checks container health |

---

## Key Distinctions to Remember

**CMD vs ENTRYPOINT**
- `CMD` — default args, easily overridden: `docker run myapp python main.py`
- `ENTRYPOINT` — locked executable, args append to it: `ENTRYPOINT ["node"]` + `CMD ["server.js"]`
- Together: ENTRYPOINT sets the binary, CMD sets defaults — best of both worlds

**COPY vs ADD**
- `COPY` — simple, explicit; copies files/dirs only — prefer this
- `ADD` — superset of COPY; also extracts `.tar.gz` and fetches URLs — use only when needed

**Named Volume vs Bind Mount**
- Named volume: Docker manages the storage (`-v mydata:/app/data`) — use for databases, persistence
- Bind mount: Host path mirrored into container (`-v $(pwd):/app`) — use for dev hot-reload

**`-p 8080:80`**
- Format: `HOST_PORT:CONTAINER_PORT`
- Requests to `localhost:8080` are forwarded to port `80` inside the container

---

## Useful docker-compose.yml Skeleton

```yaml
version: "3.9"

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    volumes:
      - ./src:/app/src
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-net

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-net

volumes:
  pgdata:

networks:
  app-net:
```

---

## Multi-Stage Build Skeleton

```dockerfile
# Stage 1 — build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2 — production image (lean)
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```