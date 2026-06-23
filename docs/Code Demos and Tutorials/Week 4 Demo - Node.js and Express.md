---
share_cis4004: "true"
site-folder: docs/Code Demos and Tutorials/
---

# Lab 1: Your First Express Server — Live Coding Session

**Duration:** 60 minutes
**Format:** Instructor live-codes; students follow along
**Prerequisite:** Week 3 (Node.js Fundamentals), Node.js LTS installed

---

## Instructor Overview

Students build a working Express server from nothing. No starter code. No cloning a repo. The goal is to feel every step — `npm init`, installing a package, writing a route, seeing it respond. MongoDB is introduced at the end as a preview for Week 5.

**End state:** A running Express server with multiple routes, JSON responses, route parameters, body parsing, and a `.env`-based config. Students leave with a project they'll continue building.

---

## Setup Checklist (Before Lab Starts)

Students should have:
- [ ] Node.js LTS installed — `node --version` (should be v18 or v20)
- [ ] npm installed — `npm --version`
- [ ] A code editor (VS Code preferred)
- [ ] A terminal they're comfortable with
- [ ] curl or a REST client (Postman, Insomnia, Bruno, or Thunder Client for VS Code)

Instructor should have:
- [ ] Empty directory ready to build in live
- [ ] Terminal and editor on projector
- [ ] Browser open to npm docs in case of questions

---

## Part 1 — Initialize the Project (8 min)

### Start from absolute zero

```bash
mkdir my-api
cd my-api
```

```bash
npm init -y
```

> "This creates `package.json`. The `-y` flag accepts all defaults. Let's look at what it created."

```bash
cat package.json
```

Show each field. Explain `main` (entry point) and `scripts` (we'll use these heavily).

**Update the start script:**
```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "echo \"No tests yet\""
  }
}
```

**Install Express:**
```bash
npm install express
```

> "Watch what happened: `package.json` now has a `dependencies` section, and `node_modules/` appeared. Also `package-lock.json` — we talked about this in lecture. Don't touch it, don't gitignore it."

**Set up .gitignore:**
```bash
echo "node_modules/" > .gitignore
echo ".env" >> .gitignore
```

**Initialize a git repo:**
```bash
git init
git add .
git commit -m "Initial project setup"
```

> "We commit now, before writing a single line of application code. This is our clean baseline."

---

## Part 2 — The Simplest Possible Server (7 min)

### Get something running before adding complexity

```bash
touch server.js
```

```javascript
// server.js
const express = require('express');

const app = express();

app.get('/', (req, res) => {
  res.send('Hello, world!');
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

```bash
node server.js
```

Open `http://localhost:3000` in a browser — show `Hello, world!`.

Then switch to curl:
```bash
curl http://localhost:3000
```

> "Two ways to hit your server: browser and curl. You'll use curl constantly. It's scriptable and shows you exactly what the server returns."

**Change `res.send` to `res.json`:**
```javascript
app.get('/', (req, res) => {
  res.json({ message: 'Hello, world!', status: 'ok' });
});
```

Restart the server (`Ctrl+C`, `node server.js`). Hit it again.

> "JSON is the language of APIs. `res.json()` sets the right `Content-Type` header automatically."

**Check-in:** Everyone should have a running server. Raise your hand if you see the JSON response.

---

## Part 3 — Add dotenv and Environment Variables (5 min)

```bash
npm install dotenv
touch .env
touch .env.example
```

```bash
# .env
PORT=3000
NODE_ENV=development
```

```bash
# .env.example
PORT=3000
NODE_ENV=development
```

```javascript
// server.js — add at the very top, before anything else
require('dotenv').config();

const express = require('express');
const app = express();

const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello, world!', status: 'ok' });
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

```bash
node server.js
```

> "Notice the port now comes from `.env`. Change it to 4000 in `.env`, restart — the server moves ports. This is how config works without touching application code."

Change it back to 3000.

---

## Part 4 — Install Nodemon (3 min)

```bash
npm install --save-dev nodemon
```

```bash
npm run dev
```

Make a change to the response — save the file. Show the server restarting automatically.

> "No more `Ctrl+C` + `node server.js` every time. Nodemon watches for file changes and restarts. It's a `devDependency` — it never goes to production."

---

## Part 5 — Build Multiple Routes (12 min)

### A fake in-memory data store

> "We're not connecting to a database yet — that's Lab 2. We'll use a JavaScript array as our data source. Same routes, same patterns, no setup required."

```javascript
// server.js
require('dotenv').config();

const express = require('express');
const app = express();

app.use(express.json());  // parse JSON request bodies

const PORT = process.env.PORT || 3000;

// --- In-memory data store ---
let users = [
  { id: 1, name: 'Alice', email: 'alice@example.com', role: 'admin' },
  { id: 2, name: 'Bob', email: 'bob@example.com', role: 'user' },
  { id: 3, name: 'Carol', email: 'carol@example.com', role: 'user' },
];
let nextId = 4;

// --- Routes ---

// GET /users — return all users
app.get('/users', (req, res) => {
  res.json(users);
});

// GET /users/:id — return one user
app.get('/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const user = users.find(u => u.id === id);

  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }

  res.json(user);
});

// POST /users — create a new user
app.post('/users', (req, res) => {
  const { name, email, role } = req.body;

  if (!name || !email) {
    return res.status(400).json({ error: 'Name and email are required' });
  }

  const newUser = { id: nextId++, name, email, role: role || 'user' };
  users.push(newUser);

  res.status(201).json(newUser);
});

// PATCH /users/:id — update a user
app.patch('/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const userIndex = users.findIndex(u => u.id === id);

  if (userIndex === -1) {
    return res.status(404).json({ error: 'User not found' });
  }

  users[userIndex] = { ...users[userIndex], ...req.body, id };
  res.json(users[userIndex]);
});

// DELETE /users/:id — delete a user
app.delete('/users/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const userIndex = users.findIndex(u => u.id === id);

  if (userIndex === -1) {
    return res.status(404).json({ error: 'User not found' });
  }

  users.splice(userIndex, 1);
  res.status(204).send();
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

**Test each route live with curl:**

```bash
# GET all users
curl http://localhost:3000/users

# GET one user
curl http://localhost:3000/users/1

# GET a user that doesn't exist
curl http://localhost:3000/users/99
# → 404

# POST a new user
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Dave","email":"dave@example.com"}'

# POST with missing fields
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Incomplete"}'
# → 400

# PATCH a user
curl -X PATCH http://localhost:3000/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Updated"}'

# DELETE a user
curl -X DELETE http://localhost:3000/users/2
# → 204 no content

# Confirm deletion
curl http://localhost:3000/users
```

**Point out patterns:**
- Every route checks for existence and returns 404 before operating
- POST validates input and returns 400 for bad requests
- POST returns `201 Created`, not `200`
- DELETE returns `204 No Content` — no body

**Check-in:** Have everyone curl `POST /users` and create themselves as a user. Show the result in `GET /users`.

---

## Part 6 — Query Parameters (5 min)

### Filter by role

Add this before the other `GET /users` route (order matters in Express):

```javascript
// GET /users?role=admin
app.get('/users', (req, res) => {
  const { role } = req.query;

  if (role) {
    const filtered = users.filter(u => u.role === role);
    return res.json(filtered);
  }

  res.json(users);
});
```

```bash
curl "http://localhost:3000/users?role=admin"
curl "http://localhost:3000/users?role=user"
curl http://localhost:3000/users   # all users, no filter
```

> "Query parameters are optional. If `role` isn't in the URL, we return all users. If it is, we filter. Always treat query params as optional."

---

## Part 7 — Extract Routes to a Router (5 min)

### Clean up server.js before it gets out of hand

```bash
mkdir routes
touch routes/users.js
```

Move all user routes into `routes/users.js`:

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

let users = [
  { id: 1, name: 'Alice', email: 'alice@example.com', role: 'admin' },
  { id: 2, name: 'Bob', email: 'bob@example.com', role: 'user' },
  { id: 3, name: 'Carol', email: 'carol@example.com', role: 'user' },
];
let nextId = 4;

router.get('/', (req, res) => {
  const { role } = req.query;
  if (role) return res.json(users.filter(u => u.role === role));
  res.json(users);
});

router.get('/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

router.post('/', (req, res) => {
  const { name, email, role } = req.body;
  if (!name || !email) return res.status(400).json({ error: 'Name and email are required' });
  const newUser = { id: nextId++, name, email, role: role || 'user' };
  users.push(newUser);
  res.status(201).json(newUser);
});

router.patch('/:id', (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'User not found' });
  users[idx] = { ...users[idx], ...req.body, id: users[idx].id };
  res.json(users[idx]);
});

router.delete('/:id', (req, res) => {
  const idx = users.findIndex(u => u.id === parseInt(req.params.id));
  if (idx === -1) return res.status(404).json({ error: 'User not found' });
  users.splice(idx, 1);
  res.status(204).send();
});

module.exports = router;
```

```javascript
// server.js — now clean
require('dotenv').config();

const express = require('express');
const usersRouter = require('./routes/users');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());
app.use('/users', usersRouter);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

> "server.js is back to being readable. Routes live where they belong. Adding a `posts` resource means adding `routes/posts.js` and one line in server.js."

---

## Part 8 — Stretch Goal: Simple Middleware (5 min, if time permits)

```bash
mkdir middleware
touch middleware/logger.js
```

```javascript
// middleware/logger.js
function logger(req, res, next) {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} ${res.statusCode} - ${duration}ms`);
  });
  next();
}

module.exports = logger;
```

```javascript
// server.js
const logger = require('./middleware/logger');
app.use(logger);  // add before routes
```

Hit a few routes and watch the logs:
```
GET /users 200 - 3ms
GET /users/99 404 - 1ms
POST /users 201 - 2ms
```

---

## Wrap-Up and Commit (5 min)

```bash
git add .
git commit -m "Add Express server with user routes and dotenv"
```

### What you built today

- ✅ Node project initialized with `npm init`
- ✅ Express installed and running
- ✅ Environment variables via dotenv
- ✅ Nodemon for dev workflow
- ✅ Full CRUD routes: GET, POST, PATCH, DELETE
- ✅ Route parameters (`req.params`), query strings (`req.query`), request body (`req.body`)
- ✅ Appropriate status codes: 200, 201, 204, 400, 404
- ✅ Routes extracted to Express Router
- ✅ (Stretch) Custom request logging middleware

### Before Lab 2

- Install MongoDB locally: [mongodb.com/try/download/community](https://www.mongodb.com/try/download/community)
  - OR use MongoDB Atlas (free cloud tier) — no local install needed
- Run `mongosh` to confirm it works
- Think about what your project's main resources are (what "things" does your app manage?)

---

## Troubleshooting Guide

### "Cannot find module 'express'"
`npm install` wasn't run, or was run in the wrong directory. Confirm `node_modules/` exists in the project root.

### "EADDRINUSE: address already in use"
Port 3000 is already taken. Either kill the old server (`Ctrl+C`) or change `PORT` in `.env` to `3001`.

### "SyntaxError: Unexpected token" in server.js
JSON parse error in a request body, usually from a malformed `curl` command. Check that the `-d` body is valid JSON with double quotes.

### "req.body is undefined"
`app.use(express.json())` is missing or placed after the route. It must come before any routes that read `req.body`.

### "curl: (3) URL using bad/illegal format"
The URL has special characters. Wrap it in quotes: `curl "http://localhost:3000/users?role=admin"`.

### Route returns 404 for everything
Check that the router is mounted: `app.use('/users', usersRouter)`. If the router file path is wrong, Node will throw a module not found error on startup.

### nodemon not found
It was installed as a devDependency but not globally. Use `npm run dev` (which uses the local installation via the script), not `nodemon server.js` directly.
