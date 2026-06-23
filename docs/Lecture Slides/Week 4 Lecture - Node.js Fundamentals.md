---
theme: css/ucf-knights.css
height: "1080"
width: "1920"
share_cis4004: "true"
share_cop4331c: "true"
site-folder: docs/Lecture Slides
---

# Node.js Fundamentals
### JavaScript, outside the browser

*"Node didn't invent server-side JavaScript. It made it worth using."*

---

## What Is Node.js?

### JavaScript, but on the server

Before Node (pre-2009):
- JavaScript ran only in browsers
- Servers used PHP, Ruby, Java, Python
- Frontend devs had to context-switch to a different language for backend work

---

**Node.js** is a JavaScript runtime built on Chrome's V8 engine. It lets you run JavaScript anywhere — on a server, your laptop, a Raspberry Pi.

What Node is **not:**
- Not a web framework (that's Express)
- Not a language (still JavaScript)
- Not just for web servers — CLI tools, scripts, build tools, desktop apps

> Node.js made JavaScript a general-purpose language.

---

## Why Node for Backend?

**One language, full stack**
- Write JavaScript on the frontend and backend
- Share validation logic, types, and utilities between client and server
- Smaller context switch for developers

**Massive ecosystem**
- npm has over 2 million packages
- If you need it, a package probably exists

**Fast for I/O-heavy work**
- APIs, databases, file systems — Node handles these efficiently
- The reason is the event loop (next slide)

**Used in production by:**
Netflix, LinkedIn, Uber, PayPal, NASA, GitHub

---

## The Event Loop: The Core Idea

### Why Node can handle many requests without many threads

- JavaScript is single-threaded, but can hand off some tasks to browser or Node APIs
- These APIs do the work while JavaScript moves on to other tasks

---

### Traditional servers (PHP, Java): one thread per request
```
Request 1 → Thread 1 (waiting for DB...)
Request 2 → Thread 2 (waiting for DB...)
Request 3 → Thread 3 (waiting for DB...)
1000 requests → 1000 threads → out of memory
```
---

### Node: one thread, event-driven
```
Request 1 → ask DB for data → go handle other things
Request 2 → ask DB for data → go handle other things
Request 3 → ask DB for data → go handle other things
DB responds to Request 1 → handle it
DB responds to Request 2 → handle it
```

**Node doesn't wait. It delegates and moves on.**

---

## The Event Loop: How It Works

### The simplified mental model

```
┌─────────────────────────────────┐
│           Call Stack            │  ← your synchronous code runs here
└─────────────────┬───────────────┘
                  │ when stack is empty
                  ▼
┌─────────────────────────────────┐
│          Event Queue            │  ← completed async tasks wait here
│  [DB result] [file read] [timer]│
└─────────────────┬───────────────┘
                  │ event loop picks next item
                  ▼
             Back to call stack
```

**The event loop:** constantly checks — "is the call stack empty? is there something in the queue?" If yes to both: process the next queued item.

This is why Node is **single-threaded but non-blocking**.

---

## Synchronous vs Asynchronous

### The most important Node.js concept to internalize

**Synchronous — blocks everything:**
```javascript
const fs = require('fs');
  // nothing else runs while waiting
const data = fs.readFileSync('bigfile.txt');
console.log(data);
console.log('this runs after the file is read');
```
---

**Asynchronous — non-blocking:**
```javascript
const fs = require('fs');
// register a callback, move on
fs.readFile('bigfile.txt', (err, data) => {  
  console.log(data);      // runs when file is ready
});
console.log('this runs immediately,'); 
console.log('before the file is read');
```

**Rule:** In Node, any operation that touches I/O (files, network, database) should be async. Using sync versions in a server blocks every request while one waits.

---

## Async Patterns: Callbacks → Promises → Async/Await

### JavaScript async has evolved — know all three
1. Callbacks
2. Promises
3. Async/Await


---

**Callbacks (oldest, still common in Node core):**
```javascript
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log(data);
});
```
---

**Promises:**
```javascript
const { readFile } = require('fs/promises');

readFile('file.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.error(err));
```
---

**Async/Await (modern, preferred):**
```javascript
const { readFile } = require('fs/promises');

async function loadFile() {
  try {
    const data = await readFile('file.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

> You'll use async/await for almost everything in Express. Know callbacks because you'll read them in docs and older code.

---

## Node Modules: CommonJS (Legacy)

### How Node organizes and shares code

---

**Exporting from a file:**
```javascript
// math.js
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

module.exports = { add, subtract };
```

---

**Importing in another file:**
```javascript
// app.js
const { add, subtract } = require('./math');

console.log(add(2, 3));       // 5
console.log(subtract(10, 4)); // 6
```
---

**Importing a built-in Node module:**
```javascript
const path = require('path');
const http = require('http');
const fs = require('fs');
```

**Importing an npm package:**
```javascript
const express = require('express');  // after npm install express
```

---
## Node Modules: ES Modules

---

```javascript
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
```

---

## npm: The Package Manager

---

### npm does three things:

1. Install packages
2. Run scripts
3. Manage versions/updates

---

**1. Install packages**
```bash
npm install express          # add to dependencies
npm install --save-dev jest  # add to devDependencies (not in production)
npm install                  # install everything in package.json
npm ci                       # clean install (use in CI/CD)
```

---

**2. Run scripts**
```bash
npm start    # node server.js (or whatever you configure)
npm test     # run your test suite
npm run dev  # any custom script you define
```

---

**3. Manage versions**
```bash
npm init -y              # create package.json
npm update               # update packages
npm outdated             # see what's out of date
```

---

### package.json: The Project Blueprint

```json
{
  "name": "my-api",
  "version": "1.0.0",
  "description": "A simple REST API",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.19.2",
    "mongoose": "^8.4.0"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "nodemon": "^3.1.0"
  }
}
```

**`dependencies`** — needed to run the app
**`devDependencies`** — only needed during development/testing
**`scripts`** — shortcuts for common commands
**`^`** — means "compatible with this major version"

---

## package-lock.json: Why It Matters

### The file you shouldn't ignore (or edit manually)

- `package.json` says: "I need express version `^4.19.0`"
- `package-lock.json` says: "I installed express `4.19.2`, which depends on X `1.2.3`, which depends on Y `0.8.1`..."

**Why this matters:**
- `npm install` on two different machines might install different patch versions
- `package-lock.json` pins the exact version tree
- `npm ci` uses `package-lock.json` strictly — always reproducible

**Rules:**
- ✅ Commit `package-lock.json` to version control
- ✅ Use `npm ci` in production and CI
- ❌ Never edit it manually
- ❌ Never add it to `.gitignore`

---

### Your First Node.js Server (Without Express)

```javascript
// server-raw.js
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.url === '/' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello, world!');
  } else {
    res.writeHead(404, { 'Content-Type': 'text/plain' });
    res.end('Not found');
  }
});

server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

```bash
node server-raw.js
curl http://localhost:3000
# → Hello, world!
```

This works. But routing, parsing request bodies, setting headers — it all has to be done manually. **Express handles all of this.**

---

## Enter Express

### Express is a minimal, fast web framework for Node

Express gives you:
- **Routing** — map URLs and HTTP methods to functions
- **Middleware** — a pipeline for processing requests
- **Request/Response helpers** — `res.json()`, `req.body`, `req.params`
- **Error handling** — a standard way to catch and respond to errors

---

```bash
npm install express
```

```javascript
// server.js
const express = require('express');
const app = express();

app.use(express.json());  // parse JSON request bodies

app.get('/', (req, res) => {
  res.json({ message: 'Hello, world!' });
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

**That's it.** Compare to the raw http version — Express handles the boilerplate.

---

## The Request/Response Cycle in Express

### What happens when a request comes in

```
Client sends: GET /users/42
                    │
                    ▼
         ┌──────────────────────────┐
         │   Express Router         │
         │  matches GET /users/:id  │
         └──────────┬───────────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │    Middleware       │  ← runs in order (auth, logging, etc.)
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │   Route Handler     │  ← your function runs here
         │   (req, res) => {}  │
         └──────────┬──────────┘
                    │
                    ▼
         Client receives: 200 { id: 42, name: "Alice" }
```

---

### Every Express handler receives two objects: 
- **`req`** (the incoming request) 
- **`res`** (your outgoing response).

---

## Nodemon: Restart on Change

### Stop restarting your server manually!

---

### Without nodemon:
```bash
node server.js
# make a change
Ctrl+C
node server.js  # repeat forever
```

---

### With nodemon:
```bash
npm install --save-dev nodemon
```

```json
// package.json
"scripts": {
  "dev": "nodemon server.js"
}
```

```bash
npm run dev
# Server restarts automatically on every file save
```

> Nodemon is a development tool only. In production you use a process manager like pm2. In CI you use `node` directly.

---

## Environment Variables with `dotenv`

### Configuration that isn't hardcoded

---

## Keep Your Secrets Safe

**Why:** port numbers, database URLs, API keys change between development and production. Hardcoding them is fragile and a security risk.

```bash
npm install dotenv
```
---

## The `.env` file

```bash
# .env  (never commit this)
PORT=3000
NODE_ENV=development
MONGO_URI=mongodb://localhost:27017/myapp
```
---

## Using the `.env` file

```javascript
// server.js — load at the very top
require('dotenv').config();

const PORT = process.env.PORT || 3000;
const MONGO_URI = process.env.MONGO_URI;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## Keep it Secret!



**Add to `.gitignore`:**
```
.env
node_modules/
```

---

## What's Coming in the Demo

### Build Our First Express Server

You will:
1. Initialize a new Node project with `npm init`
2. Install Express and dotenv
3. Build a server with multiple routes
4. Use `req.params` and `req.body`
5. Return JSON responses with appropriate status codes
6. *(Stretch)* Add nodemon and wire up a dev script

**End state:** a running Express API you can hit with a browser or `curl`

**Come with:** Node.js installed (`node --version` should work), a code editor

> Install Node.js: [nodejs.org](https://nodejs.org) — use the LTS version

---

## Summary

### What we covered

- **Node.js** is a JavaScript runtime — JS outside the browser
- **The event loop** makes Node non-blocking: it delegates I/O and moves on
- **Async/await** is the modern pattern for handling asynchronous operations
- **CommonJS modules** let you split code across files with `require` / `module.exports`
- **npm** manages packages, scripts, and versioning
- **package.json** defines the project; **package-lock.json** locks exact versions
- **Express** is a minimal framework that wraps Node's http module
- **dotenv** keeps configuration out of your source code

**Next up:** Lecture 2 — REST APIs with Express and MongoDB

*Before Lab 1:* confirm `node --version` and `npm --version` work in your terminal.

---

## Speaker Notes

### Slide 4–5 (Event Loop)
Don't go deep on the internals (microtask queue, libuv, etc.) — that's a rabbit hole. The mental model that matters: Node doesn't sit and wait; it registers a callback and goes back to work. Ask: "What would happen to a restaurant if one waiter had to stand at the kitchen window waiting for every order before taking another table?" That's threaded blocking I/O. Node is the waiter who drops off the order, takes three more tables, and comes back when the kitchen calls.

### Slide 6 (Sync vs Async)
Run both examples live if possible. The output order surprises students every time — "this runs immediately" printing before the file contents is counterintuitive until the event loop clicks.

### Slide 11 (package-lock.json)
This is where students make their first git mistakes. They add it to .gitignore, or they delete it and wonder why `npm ci` fails in CI. Spend 2 minutes here. It will save you debugging sessions later.

### Slide 12 (Raw HTTP Server)
Show this briefly — 60 seconds — then move to Express. The point isn't to teach the http module; it's to motivate why Express exists. "You could do all of this manually. Express means you don't have to."
