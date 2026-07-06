---
share_cis4004: "true"
site-folder: docs/Code Demos and Tutorials/
theme: ucf-knights.css
height: "1080"
width: "1920"
---
## Instructor Overview

This lab builds a Docker-based development environment for students' Node.js projects. The goal is a workflow where any teammate can clone the repo and be running with `docker compose up` — no setup doc required.

**End state:** Every student has:
- A `Dockerfile` for their Node.js app
- A `docker-compose.yml` with app + database
- Bind mounts for live code reloading in dev
- `.env.example` and `.gitignore` updated
- (Stretch) Docker service wired into their Lab 1 CI workflow

---

## Setup Checklist (Before Lab Starts)

Students should have:
- [ ] Docker Desktop installed and running (`docker run hello-world` works)
- [ ] Their Node.js project repo checked out locally
- [ ] Know what database their app uses (Postgres, MySQL, MongoDB, or none)
- [ ] Editor open

Instructor should have:
- [ ] Docker Desktop running
- [ ] Demo Node.js app ready (simple Express app with a DB connection works well)
- [ ] Terminal visible on projector

---

## Part 1 — Confirm Docker Is Working (5 min)

### Instructor script

> "Before we write anything, let's make sure everyone is in the same place."

```bash
# Every student runs this
docker --version
docker compose version
docker run hello-world
```

Expected output from `hello-world`:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

> "If you see that, you're good. If you see an error, raise your hand now — we'll fix it before moving on."

**Common fix for "permission denied" on Linux:**
```bash
sudo usermod -aG docker $USER
# Log out and back in
```

**Common fix for "Docker Desktop not running":** Open Docker Desktop app, wait for whale icon in menu bar to stop animating.

---

## Part 2 — Write the Dockerfile (15 min)

### Start from scratch, explain every line

> "We're going to write a Dockerfile together. This tells Docker exactly how to build an image of your app."

```bash
# In the root of the project
touch Dockerfile
```

**Type this live, pause and explain each instruction:**

```dockerfile
FROM node:20-alpine
```
> "We're starting from an official Node.js image. Alpine is a minimal Linux — about 5MB vs 900MB for the full image. Same Node.js, much smaller."

```dockerfile
WORKDIR /app
```
> "All subsequent commands run from this directory inside the container. Think of it as `cd /app` but it also creates the directory."

```dockerfile
COPY package*.json ./
```
> "Copy ONLY the package files first. We haven't copied source code yet. Why? The next step is slow."

```dockerfile
RUN npm ci
```
> "Install dependencies. This layer gets cached — it only re-runs if package.json changes. If we copied all our source code first, every code change would re-run this."

```dockerfile
COPY . .
```
> "NOW copy the rest of the source code. This layer is fast to invalidate and rebuild."

```dockerfile
EXPOSE 3000
```
> "Documents which port the app uses. Doesn't actually open anything — that happens at `docker run`."

```dockerfile
CMD ["node", "server.js"]
```
> "The command that starts the app. Use the array form — it's more reliable than a string."

**Full Dockerfile:**
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### Build and test it

```bash
docker build -t myapp:dev .
```

Walk through the output — point out each layer, show the caching indicator.

```bash
docker run -p 3000:3000 myapp:dev
```

> "Open localhost:3000 in your browser. Your app is running inside a container."

**Show the cache working:**
```bash
# Make a trivial code change (add a comment)
# Rebuild
docker build -t myapp:dev .
```

> "See how steps 1-4 say 'CACHED'? Only the COPY and CMD steps re-ran. That's the layer cache saving you time."

```bash
# Stop the container
Ctrl+C
```

---

## Part 3 — Add .dockerignore (5 min)

```bash
touch .dockerignore
```

```
node_modules
.git
.env
*.log
.DS_Store
npm-debug.log*
coverage/
```

> "The most important line is `node_modules`. Your Mac's node_modules can't run on Linux. We always install fresh inside the container with `npm ci`. Without this, Docker copies 200MB of platform-specific binaries into the image and then npm ci overwrites them anyway."

**Show the difference:**
```bash
# Check image size before .dockerignore
docker images myapp

# Add .dockerignore, rebuild
docker build -t myapp:dev .
docker images myapp
```

> "Likely went from ~400MB to ~200MB just from excluding node_modules."

---

## Part 4 — Set Up Environment Variables (5 min)

```bash
# Create .env.example (commit this)
touch .env.example
```

```bash
# .env.example
PORT=3000
NODE_ENV=development
DATABASE_URL=postgres://user:password@db:5432/myapp
```

```bash
# Create .env (DON'T commit this)
cp .env.example .env
# Edit .env with real values if needed
```

**Update .gitignore:**
```bash
echo ".env" >> .gitignore
```

> "`.env.example` is your documentation — it tells teammates what variables the app needs without exposing real values. `.env` has the real values and never leaves your machine."

---

## Part 5 — Write docker-compose.yml (18 min)

> "Running a single container is fine, but most apps need more than one process. Your app needs a database. `docker compose` lets us define the whole stack in one file."

```bash
touch docker-compose.yml
```

**Build this up live:**

```yaml
version: '3.8'
```
> "Specifies the Compose file format version."

```yaml
services:
  app:
    build: .
```
> "The `app` service builds from the Dockerfile in the current directory."

```yaml
    ports:
      - "3000:3000"
```
> "Map port 3000 on your machine to port 3000 in the container. Left side is your machine, right side is the container."

```yaml
    env_file:
      - .env
```
> "Load environment variables from our .env file."

```yaml
    volumes:
      - .:/app
      - /app/node_modules
```
> "Two volumes here. The first mounts our local code into the container — any file you save locally immediately appears inside. The second is a trick: it tells Docker to use the container's node_modules, not the one from your local mount."

```yaml
    depends_on:
      - db
```
> "Wait for the db service to start before starting the app."

```yaml
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```
> "The db service uses the official Postgres image — no Dockerfile needed. We persist the data in a named volume so the database survives container restarts. We also expose port 5432 so you can connect with a GUI like TablePlus."

**Adjust for students' databases:**
- MongoDB: `image: mongo:7`, port `27017`, env `MONGO_INITDB_DATABASE: myapp`
- MySQL: `image: mysql:8`, port `3306`, env `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`
- No database: skip the `db` service and `volumes` section

**Full docker-compose.yml:**
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    env_file:
      - .env
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```

### Run it

```bash
docker compose up --build
```

> "Watch the output. You'll see both services start. The first run downloads the Postgres image — that's a one-time thing."

Check it works:
- App: `curl http://localhost:3000` or open in browser
- DB: use a DB GUI or `docker compose exec db psql -U user myapp`

**Show live reloading:**
```javascript
// Make a change to a route or response
// Save the file
// (If using nodemon) Watch the app restart in the compose output
// Curl the endpoint again — change is reflected
```

> "No rebuild. The bind mount means your local file system IS the container's file system."

---

## Part 6 — Key Compose Commands (5 min)

```bash
# Run in background
docker compose up -d

# View logs
docker compose logs -f app

# Run a command inside a running service
docker compose exec app sh           # open a shell
docker compose exec app npm test     # run tests inside container

# Stop everything
docker compose down

# Stop AND wipe the database volume
docker compose down -v

# Rebuild after Dockerfile changes
docker compose up --build
```

> "Commit this to memory: `up`, `down`, `exec`, `logs`. That's 90% of what you'll use day to day."

---

## Part 7 — Update .gitignore and Commit (2 min)

```bash
# Confirm .gitignore has the important exclusions
cat .gitignore
```

Should include:
```
node_modules/
.env
```

```bash
git add Dockerfile .dockerignore docker-compose.yml .env.example .gitignore
git commit -m "Add Docker development environment"
git push
```

> "Notice what we're NOT committing: `.env`, `node_modules`. What we ARE committing: `Dockerfile`, `docker-compose.yml`, `.env.example`. The setup is reproducible for any teammate."

---

## Part 8 — Stretch Goal: Docker in CI (10 min, if time permits)

> "Remember the CI workflow from Lab 1? Let's make the test job use a Postgres service — same version as our docker-compose."

**Open `.github/workflows/ci.yml` and update the `test` job:**

```yaml
  test:
    runs-on: ubuntu-latest
    needs: lint

    services:
      db:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: password
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - run: npm test
        env:
          DATABASE_URL: postgres://user:password@localhost:5432/testdb
          NODE_ENV: test
```

> "This spins up a real Postgres container in CI — same image as your docker-compose. Your tests now run against the same database version in CI as in development. That's CI/CD parity."

```bash
git add .github/workflows/ci.yml
git commit -m "Add Postgres service to CI workflow"
git push
```

---

## Wrap-Up and Debrief (5 min)

### What you built today

- ✅ `Dockerfile` — reproducible build of the Node.js app
- ✅ `.dockerignore` — lean images, no accidentally shipped node_modules
- ✅ `.env.example` — documented configuration for teammates
- ✅ `docker-compose.yml` — full dev stack in one command
- ✅ Bind mounts — live code reloading without rebuilds
- ✅ (Stretch) CI uses the same Postgres version as development

### The combined workflow students now have

```
git clone repo
cp .env.example .env        # fill in real values
docker compose up           # everything running in <2 min

# Develop locally → changes reflected instantly via bind mount
# Push to GitHub → CI runs with same database version
# PR passes CI → team can merge with confidence
```

### Discussion questions

1. "A new teammate joins your project tomorrow. Walk them through setup using what you built today."
2. "What would break if someone forgot to add `node_modules` to `.dockerignore`?"
3. "Why do we have two entries in the `volumes` list for the app service?"

---

## Troubleshooting Guide

### "Port already in use"
```bash
# Find what's using the port
lsof -i :3000   # macOS/Linux
# or just change the left port in docker-compose.yml
ports:
  - "3001:3000"
```

### "Cannot connect to Docker daemon"
Docker Desktop isn't running. Open it and wait for the whale icon to stop animating.

### "node_modules not found inside container"
Make sure the anonymous volume line is present:
```yaml
volumes:
  - .:/app
  - /app/node_modules   # ← this line
```
Without it, the bind mount overwrites the container's node_modules with your (possibly empty or wrong-platform) local one.

### "App can't connect to database"
- In docker-compose, services communicate by service name, not `localhost`. Database host should be `db` (the service name), not `localhost`.
- Make sure `DATABASE_URL` in `.env` uses `@db:5432`, not `@localhost:5432`.
- Check `depends_on: db` is in the app service.

### "Changes to code not reflected"
- Confirm the bind mount is correct: `- .:/app`
- If using nodemon, make sure the `CMD` in Dockerfile runs nodemon: `CMD ["npx", "nodemon", "server.js"]`
- Or override in docker-compose for dev:
```yaml
command: npx nodemon server.js
```

### "docker compose down doesn't delete my data"
That's intentional — named volumes persist. Use `docker compose down -v` to also delete volumes (wipes the database).

### "Image takes too long to build"
- Confirm `.dockerignore` exists and excludes `node_modules` and `.git`
- Check `cache: 'npm'` is in the setup-node step (for CI)
- Move `COPY package*.json ./` before `COPY . .`

### "I accidentally committed .env"
```bash
# Remove from tracking (keeps the local file)
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "Remove .env from tracking"
git push
```
If the repo is public or the secrets were real, rotate the credentials immediately.
