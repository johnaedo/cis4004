---
share_cis4004: "true"
site-folder: docs/Code Demos and Tutorials/
---

# Lab 2: REST API with MongoDB and Mongoose — Live Coding Session

**Duration:** 60 minutes
**Format:** Instructor live-codes; students follow along
**Prerequisites:** Week 4, Lab 1 complete, MongoDB installed or Atlas account ready

---

## Instructor Overview

Students replace the in-memory array from Lab 1 with a real MongoDB database via Mongoose. The routes they already wrote stay almost identical — the swap is in the data layer. This is intentional: it shows that good route structure is database-agnostic.

**End state:** A fully functional REST API backed by MongoDB, with a Mongoose schema, validation, error handling middleware, and a clean project structure. This is the app students will containerize and deploy in later modules.

---

## Setup Checklist (Before Lab Starts)

Students should have **one** of:
- [ ] MongoDB Community installed locally — `mongosh` works in terminal
	- [ ] Pre-Requisite:  [Install Docker Engine | Docker Docs](https://docs.docker.com/engine/install/)
- [ ] MongoDB Atlas account with a free cluster created and connection string ready

Instructor should have:
- [ ] Their Lab 1 project open (continuing from last session)
- [ ] MongoDB running locally or Atlas connection string ready
- [ ] MongoDB Compass or mongosh open for showing database state live

---

## Part 1 — MongoDB Setup and Connection (10 min)

### Option A: Local MongoDB Container

#### Install Docker Engine *or* Docker Desktop
If you're running on MacOS or Windows, you'll need to install Docker Desktop:  [Get Started | Docker](https://www.docker.com/get-started/)

On your droplet, you'll need to install Docker Engine:
[Install Docker Engine | Docker Docs](https://docs.docker.com/engine/install/)

> [!WARNING]
> You will need to upgrade both CPU and memory on your droplet if you're reusing your LAMP setup.  You will need a minimum of 2GB of memory.  DigitalOcean's pricing structure is such that an upgrade in CPU comes with an upgrade in memory.


### Option B: MongoDB Atlas (cloud, no local install)

> "Go to [cloud.mongodb.com](https://cloud.mongodb.com), create a free cluster, click Connect → Drivers → copy the connection string. Replace `<password>` with your password."

Connection string looks like:
```
mongodb+srv://username:password@cluster0.abcde.mongodb.net/myapp?retryWrites=true&w=majority
```

### Install Mongoose

```bash
npm install mongoose
```

### Create the database connection module

```bash
touch db.js
```

```javascript
// db.js
const mongoose = require('mongoose');

async function connectDB() {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log(`MongoDB connected: ${mongoose.connection.host}`);
  } catch (err) {
    console.error('MongoDB connection failed:', err.message);
    process.exit(1);
  }
}

module.exports = connectDB;
```

**Update `.env`:**
```bash
PORT=3000
NODE_ENV=development
MONGO_URI=mongodb://localhost:27017/myapi
# Atlas users: paste your connection string here instead
```

**Update `.env.example`:**
```bash
PORT=3000
NODE_ENV=development
MONGO_URI=mongodb://localhost:27017/mernDemo
```

**Wire it into server.js:**
```javascript
// server.js
require('dotenv').config();

const express = require('express');
const connectDB = require('./db');
const usersRouter = require('./routes/users');

const app = express();
const PORT = process.env.PORT || 3000;

connectDB();  // connect to MongoDB on startup

app.use(express.json());
app.use('/', (req, res, next) => {
	res.status(200).json({message: "Hello, World!"});
	next();
});
app.use('/users', usersRouter);

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

```bash
npm run dev
```

> "You should see both 'MongoDB connected' and 'Server running'. If MongoDB fails to connect, the process exits — that's intentional. A server that can't reach its database shouldn't pretend to be healthy."

**Check-in:** Everyone should see both success messages in the terminal.

---

## Part 2 — Define the Mongoose Schema and Model (10 min)

```bash
mkdir models
touch models/User.js
```

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
      minlength: [2, 'Name must be at least 2 characters'],
    },
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
      trim: true,
      match: [/^\S+@\S+\.\S+$/, 'Please provide a valid email'],
    },
    role: {
      type: String,
      enum: {
        values: ['user', 'admin'],
        message: 'Role must be either user or admin',
      },
      default: 'user',
    },
  },
  {
    timestamps: true,  // adds createdAt and updatedAt
  }
);

// Don't return __v in query results
userSchema.set('toJSON', {
  transform: (doc, ret) => {
    delete ret.__v;
    return ret;
  },
});

const User = mongoose.model('User', userSchema);

module.exports = User;
```

> "The schema is where your data contract lives. If a document doesn't match, Mongoose rejects it before it ever touches the database. Compare this to our Lab 1 array — we had to validate manually. Mongoose handles it for us."

**Walk through each field:**
- `required` — validation error if missing
- `unique` — MongoDB creates an index; duplicate emails get a `11000` error code
- `enum` — only allows specific values
- `trim` / `lowercase` — data normalization happens automatically
- `timestamps: true` — free `createdAt` and `updatedAt` on every document

---

## Part 3 — Build the Error Handler Middleware (5 min)

> "Before we rewrite the routes, let's put the error handler in place. Every route will use it."

```bash
mkdir middleware
touch middleware/errorHandler.js
```

```javascript
// middleware/errorHandler.js
function errorHandler(err, req, res, next) {
  console.error(err.stack);

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const messages = Object.values(err.errors).map(e => e.message);
    return res.status(400).json({ error: messages.join(', ') });
  }

  // Duplicate key (e.g. unique email)
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    return res.status(409).json({
      error: `A user with that ${field} already exists`,
    });
  }

  // Bad MongoDB ObjectId format
  if (err.name === 'CastError') {
    return res.status(400).json({ error: 'Invalid ID format' });
  }

  // Default
  res.status(err.status || 500).json({
    error: err.message || 'Internal server error',
  });
}

module.exports = errorHandler;
```

**Register it in server.js (must be last):**
```javascript
// server.js
const errorHandler = require('./middleware/errorHandler');

// ... all other middleware and routes ...

app.use(errorHandler);  // last middleware
```

> "Four error types, all handled centrally. Every route just calls `next(err)` and this middleware takes over. You write this once, use it everywhere."

---

## Part 4 — Rewrite the Routes with Mongoose (18 min)

> "Now the interesting part. We're going to replace the in-memory array operations with Mongoose. Notice how similar the route structure stays — the only thing changing is the data layer."

**Rewrite `routes/users.js` from scratch:**

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');

// GET /users  (optionally filter by role: ?role=admin)
router.get('/', async (req, res, next) => {
  try {
    const filter = {};
    if (req.query.role) filter.role = req.query.role;

    const users = await User.find(filter).sort({ createdAt: -1 });
    res.json(users);
  } catch (err) {
    next(err);
  }
});

// GET /users/:id
router.get('/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
  } catch (err) {
    next(err);  // CastError (bad ID format) is caught here → errorHandler
  }
});

// POST /users
router.post('/', async (req, res, next) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    next(err);  // ValidationError and duplicate key caught here → errorHandler
  }
});

// PATCH /users/:id
router.patch('/:id', async (req, res, next) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
  } catch (err) {
    next(err);
  }
});

// DELETE /users/:id
router.delete('/:id', async (req, res, next) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.status(204).send();
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

> "Compare this to Lab 1. The routes look almost identical. `users.find(u => u.id === id)` became `User.findById(id)`. The `try/catch + next(err)` pattern is consistent on every handler. This is what good API structure looks like."

---

## Part 5 — Test Every Route (10 min)

**Work through each route live:**

```bash
# Create users
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","role":"admin"}'

curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Bob","email":"bob@example.com"}'

# List all users
curl http://localhost:3000/users

# Filter by role
curl "http://localhost:3000/users?role=admin"
```

**Show the MongoDB side — open mongosh or Compass:**
```javascript
// in mongosh
use myapi
db.users.find().pretty()
```

> "There they are. Real documents, in a real database, with `_id`, `createdAt`, `updatedAt`. This persists across server restarts."

```bash
# Get one user — copy an _id from the previous response
curl http://localhost:3000/users/PASTE_ID_HERE

# Test validation — missing required field
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"No Email"}'
# → 400 with validation message

# Test duplicate email
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice 2","email":"alice@example.com"}'
# → 409 conflict

# Test bad ID format
curl http://localhost:3000/users/notanid
# → 400 invalid ID format

# Update a user
curl -X PATCH http://localhost:3000/users/PASTE_ID_HERE \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Updated"}'

# Delete a user
curl -X DELETE http://localhost:3000/users/PASTE_ID_HERE

# Confirm deletion
curl http://localhost:3000/users
```

**Check-in:** Everyone should be hitting a real MongoDB database. Stop and help anyone who's still seeing connection errors.

---

## Part 6 — Final Project Structure Review (3 min)

```bash
ls -R
```

```
my-api/
├── server.js
├── db.js
├── .env               (gitignored)
├── .env.example       (committed)
├── .gitignore
├── package.json
├── package-lock.json
├── models/
│   └── User.js
├── routes/
│   └── users.js
└── middleware/
    └── errorHandler.js
```

> "This is a real project structure. Not a tutorial scaffold — something you'd actually see in a professional codebase. Every file has one clear job."

---

## Part 7 — Commit and Push (2 min)

```bash
git add .
git commit -m "Add MongoDB/Mongoose integration and error handling"
git push
```

---

## Part 8 — Stretch Goal: Add a Second Resource (remaining time)

> "If you're ahead: add a `Post` resource. Posts belong to a user."

```javascript
// models/Post.js
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: [true, 'Title is required'],
      trim: true,
    },
    body: {
      type: String,
      required: [true, 'Body is required'],
    },
    author: {
      type: mongoose.Schema.Types.ObjectId,  // reference to a User document
      ref: 'User',
      required: true,
    },
  },
  { timestamps: true }
);

module.exports = mongoose.model('Post', postSchema);
```

```javascript
// routes/posts.js — GET and POST only
const express = require('express');
const router = express.Router();
const Post = require('../models/Post');

router.get('/', async (req, res, next) => {
  try {
    const posts = await Post.find().populate('author', 'name email');
    res.json(posts);
  } catch (err) {
    next(err);
  }
});

router.post('/', async (req, res, next) => {
  try {
    const post = await Post.create(req.body);
    res.status(201).json(post);
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

```javascript
// server.js
app.use('/posts', require('./routes/posts'));
```

```bash
# Create a post (use a real user _id)
curl -X POST http://localhost:3000/posts \
  -H "Content-Type: application/json" \
  -d '{"title":"My First Post","body":"Hello world","author":"PASTE_USER_ID"}'

# List posts with author populated
curl http://localhost:3000/posts
```

> "Notice `.populate('author', 'name email')` — Mongoose looks up the referenced user and embeds their name and email in the response. One query, joined data."

---

## Wrap-Up and Debrief (5 min)

### What you built today

- ✅ MongoDB connection with graceful failure handling
- ✅ Mongoose schema with built-in validation and type coercion
- ✅ Full CRUD routes backed by a real database
- ✅ Central error handler that distinguishes validation, duplicate key, and cast errors
- ✅ A project structure ready for Docker and CI/CD
- ✅ (Stretch) Related resources with `populate`

### Discussion questions

1. "What happens if two users try to create an account with the same email at the exact same moment?"
2. "The in-memory array was simpler — why use a real database?"
3. "What would you add to the schema for your actual project?"

### What comes next

- **Docker module:** containerize this Express app, run MongoDB as a Docker service
- **CI/CD module:** run `npm test` against this API on every push to GitHub
- The project you built today is the foundation for both

---

## Troubleshooting Guide

### "MongoServerError: connect ECONNREFUSED"
MongoDB isn't running. On macOS: `brew services start mongodb-community`. On Linux: `sudo systemctl start mongod`. For Atlas: check your connection string and that your IP is whitelisted in Network Access.

### "MongoParseError: Invalid scheme"
Your `MONGO_URI` is malformed. Local should start with `mongodb://`, Atlas with `mongodb+srv://`. Check `.env` for typos.

### "Cannot destructure property of undefined" in errorHandler
The error handler must be registered with four arguments `(err, req, res, next)` — all four, even if `next` isn't used. Express uses the argument count to identify error handlers. If it's only three, Express treats it as a regular middleware and never sends errors to it.

### "req.body is undefined" in POST routes
`app.use(express.json())` must come before the routes in `server.js`. Order matters.

### ValidationError not caught — instead getting "UnhandledPromiseRejection"
The route handler is missing `try/catch` and `next(err)`. Every async route must wrap its body in try/catch and call `next(err)` in the catch block.

### "Cast to ObjectId failed" showing raw Mongoose error instead of 400
The `errorHandler` middleware isn't registered, or it's registered before the routes instead of after. It must be the last `app.use()` call in `server.js`.

### populate returns null for author
The author `_id` used when creating the post doesn't exist in the database. Grab a fresh `_id` from `GET /users` and use it in the POST body.

### Changes to the schema aren't enforced on existing documents
Mongoose validates on write, not on read. Documents created before you added a validation rule won't be re-validated unless updated. For the purposes of this lab, `docker compose down -v` (once you have Docker) or dropping the collection in mongosh resets this.
