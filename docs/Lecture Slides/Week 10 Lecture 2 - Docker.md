---
share_cis4004: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---
# Docker for Collaborative Development
### Solving "it works on my machine" once and for all

*"Ship the environment, not just the code."*

---

## The Collaboration Problem

### Your team has N machines. All different.

- Alice runs macOS with Node 20, Bob runs Windows with Node 18
- The CI server runs Ubuntu with Node 20.x (different patch version)
- The production VM runs Ubuntu 22.04
- New team member spends **half a day** just setting up their environment
- A library behaves differently on macOS vs Linux
- "It works on my machine" becomes the most dreaded phrase on the team

**This isn't a skill problem. It's an environment problem.**

---

## What Docker Actually Is

### Not a virtual machine. A contained process.

**Virtual Machine:**
- Full OS installed inside your OS
- Boots in minutes, uses gigabytes of RAM
- Complete isolation (hardware level)

**Docker Container:**
- Runs as a process on your OS kernel
- Starts in seconds, uses megabytes of RAM
- Isolation at the process/filesystem level
- Shares your kernel — no second OS to boot

**The key insight:** A container bundles your app *and its entire environment* into one portable unit. Same container runs identically on every machine.

---

## Core Concepts

### The vocabulary you need

| Concept | Analogy | What It Is |
|---------|---------|-----------|
| **Image** | A recipe | Read-only template: OS + deps + your app |
| **Container** | A dish made from the recipe | A running instance of an image |
| **Dockerfile** | The recipe instructions | Text file that builds an image |
| **Docker Hub** | npm registry for images | Public repository of pre-built images |
| **Volume** | A shared folder | Persistent storage that survives container restarts |
| **docker-compose** | A meal kit | Defines and runs multi-container setups |

One image → many containers. Just like one class → many objects.

---

## Why Docker for Collaboration (Not Just Deployment)

### You're deploying to a VM. So why learn Docker?

Because Docker solves problems *before* deployment:

1. **Onboarding** — new teammate runs `docker compose up` instead of following a 3-page setup doc
2. **Consistency** — everyone's dev environment is identical; "works on my machine" is gone
3. **Dependencies** — no more "you need PostgreSQL 14, not 16" version fights
4. **Services** — spin up a database, Redis, or mock API locally without installing anything
5. **CI parity** — your CI pipeline uses the same container as your dev environment

> Docker isn't just a deployment tool. It's a *collaboration* tool.

---

## A Dockerfile is just a recipe
<style>
.reveal pre {
	font-size:0.8em;
}
</style>

```dockerfile
# Start from an official base image
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package files first (for layer caching)
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy the rest of your app
COPY . .

# Tell Docker which port your app uses
EXPOSE 3000

# The command to start your app
CMD ["node", "server.js"]
```

**Each line is a "layer" — Docker caches them. Copy `package.json` before source code so dependency installs are cached.**

---

## Building and Running

### The core Docker workflow

```bash
# Build an image from a Dockerfile in the current directory
docker build -t myapp:latest .

# Run a container from that image
docker run -p 3000:3000 myapp:latest

# Run in the background (detached)
docker run -d -p 3000:3000 --name myapp myapp:latest

# See running containers
docker ps

# View logs from a container
docker logs myapp

# Stop and remove a container
docker stop myapp
docker rm myapp
```

**`-p 3000:3000`** means: map port 3000 on your machine to port 3000 inside the container.

---

## The Layer Cache: Why Order Matters

### Docker caches each instruction as a layer

```dockerfile
# ✅ GOOD: dependencies change less often than source code
COPY package*.json ./
RUN npm ci          # ← cached unless package.json changes
COPY . .            # ← only this and below re-run on code changes
```

```dockerfile
# ❌ BAD: copies everything first, cache never helps
COPY . .
RUN npm ci          # ← re-runs every time ANY file changes
```

**Mental model:** layers are like a stack. The moment one layer changes, all layers below it are invalidated. Put slow, stable things at the top.

---

## .dockerignore

### Don't copy things you don't need

```
# .dockerignore (works just like .gitignore)

node_modules        # rebuilt inside the container by npm ci
.git                # git history not needed at runtime
.env                # NEVER copy secrets into images
*.log
.DS_Store
coverage/
dist/               # if you rebuild inside Docker
README.md
```

**Why it matters:**
- Smaller images build faster and use less disk
- `node_modules` from your Mac can't run on Linux — always exclude it
- `.env` files with secrets should **never** be baked into an image

---

## Docker Compose for Multi-Service Dev

### Most apps have more than one process

Your project probably needs:
- Your Node.js app
- A database (PostgreSQL, MongoDB, MySQL)
- Maybe a cache (Redis)
- Maybe a background worker

**`docker-compose.yml`** defines all of these together

---

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - NODE_ENV=development
    volumes:
      - .:/app                   # mount local code into container
      - /app/node_modules        # but keep container's node_modules
    depends_on:
      - db

  db:
    image: postgres:16-alpine    # official image, no Dockerfile needed
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data   # persist data across restarts

volumes:
  pgdata:
```

---

## The Power of `docker compose up`

### One command to rule them all

```bash
# Start everything defined in docker-compose.yml
docker compose up

# Start in background
docker compose up -d

# Rebuild images (after Dockerfile changes)
docker compose up --build

# Stop everything
docker compose down

# Stop AND delete volumes (wipe the database)
docker compose down -v

# Run a one-off command in a service
docker compose exec app sh       # open shell in the app container
docker compose exec db psql -U user myapp  # open psql
```
---

## New teammate workflow
```bash
git clone https://github.com/your-team/project
cd project
docker compose up
```
That's it. App + database, running, configured correctly.

---

## Volumes: Persisting and Sharing Data

**Without a volume:** data written inside a container is lost when the container stops.

**Two kinds of volumes:**

**Named volume** — Docker manages the storage location:
```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```
Good for: databases, anything that should persist across restarts.

**Bind mount** — maps a host directory into the container:
```yaml
volumes:
  - .:/app    # your local code is live inside the container
```
Good for: development — code changes on your machine instantly appear inside the container. No rebuild needed.

---

## Environment Variables and Secrets

### Configuration without hardcoding

**In `docker-compose.yml`:**
```yaml
services:
  app:
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
```
---

**Better: use a `.env` file** (which you `.gitignore`):
```bash
# .env  (never commit this)
DATABASE_URL=postgres://user:pass@db:5432/myapp
JWT_SECRET=supersecret
```

```yaml
services:
  app:
    env_file:
      - .env
```

**Provide a `.env.example`** (do commit this):
```bash
# .env.example  (safe to commit — no real values)
DATABASE_URL=postgres://user:password@db:5432/dbname
JWT_SECRET=replace_me
```

---

## Docker in Your CI Pipeline

Your GitHub Actions workflow can use Docker too:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      db:
        image: postgres:16-alpine      # spin up postgres as a service
        env:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: pass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
```

---
## GitHub Actions Cont'd

```
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: postgres://user:pass@localhost:5432/testdb
```

**This is CI/CD parity:** same database version in CI as in development.

---

## Common Dockerfile Patterns for Node.js

### Production-ready patterns

**Multi-stage build** (smaller production image):
```dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: production (only what's needed to run)
FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev     # no devDependencies in prod
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Result:** production image doesn't contain your build tools, test frameworks, or dev dependencies.

---

## What Docker Is NOT Good For (In This Context)

### Know the tool's limits

**For your projects in this course:**

- ❌ **You are NOT deploying via Docker to your VM** — you're deploying with git pull + pm2. Docker isn't required for deployment.
- ❌ **Don't containerize your production VM** unless you want to go down a rabbit hole (orchestration, Docker Swarm, etc.)
- ❌ **Don't use Docker as a substitute for learning Node.js** — you still need to understand what's running inside

**Use Docker for:**
- ✅ Shared development environment across your team
- ✅ Running a local database without installing PostgreSQL/MySQL
- ✅ Ensuring CI runs in the same environment as dev
- ✅ Onboarding — `docker compose up` as the first step

---

## Practical Team Workflow

```
New team member joins
        │
        ▼
  git clone repo
  cp .env.example .env
  (fill in any real values)
        │
        ▼
  docker compose up --build
        │
        ▼
  App running at localhost:3000
  DB running at localhost:5432
  All services configured correctly
        │
        ▼
  Develop with bind mounts:
  Edit code locally → changes appear in container instantly
        │
        ▼
  git commit, push → CI runs in same environment
```

**Total setup time: under 10 minutes.**

---

## What's Coming in the Lab

### Demo 2 (60 min): Docker + Compose Hands-on

You will:
1. Write a `Dockerfile` for your Node.js app
2. Build and run it locally
3. Add a `docker-compose.yml` with your app + a database service
4. Use a bind mount so code changes hot-reload
5. Add a `.env.example` and update your `.gitignore`
6. *(Stretch)* Add a Docker service step to your GitHub Actions workflow from Lab 1

**Come with:** your project repo, `docker` and `docker compose` installed

> Install: [docs.docker.com/get-docker](https://docs.docker.com/get-docker/)

---

## Summary

### What we covered

- **Containers** package your app *and its environment* — no more "works on my machine"
- **Images** are built from **Dockerfiles** — recipes for reproducible environments
- **Layer caching** — order your Dockerfile instructions from stable → volatile
- **.dockerignore** — keep images small, keep secrets out
- **docker-compose** — define your full dev stack in one file
- **Volumes** — named for persistence, bind mounts for live development
- **Env files** — configure without hardcoding; `.env.example` for teammates

**The payoff:** anyone can clone your repo and be running in minutes.

---

## Speaker Notes

### Slide 3 (What Docker Actually Is)
The VM vs container distinction is commonly misunderstood. Draw both side by side. The point isn't deep OS knowledge — it's that containers start fast, use little memory, and run identically everywhere.

### Slide 5 (Why Docker for Collaboration)
Address the obvious question early: "But we're deploying to a VM, not Docker." Acknowledge it. The value is in the dev environment and CI parity, not the deployment. Students who get this use Docker well; students who don't skip the Dockerfile when things get hard.

### Slide 8 (Layer Cache)
This is the single most practical Dockerfile concept. Show both examples side by side. Ask: "How often does your package.json change vs your source code?" That ratio is exactly why order matters.

### Slide 11 (docker compose up)
If you have time, do a live demo here. Run `docker compose up` in a real project, show the output, show the app working. This lands better than any slide.

### Slide 13 (Environment Variables)
Emphasize `.env.example`. It's a small habit that saves enormous pain — every team has a story about someone committing credentials to GitHub. Make this a non-negotiable team standard.

### Slide 16 (What Docker Is NOT Good For)
This prevents scope creep. Students will ask "should we Dockerize our VM deployment?" The answer for this course is no. Keep it focused on the dev workflow use case.
