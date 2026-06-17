---
share_cis4004: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---

# REST APIs with Express and MongoDB
### Building the backend your frontend actually talks to

*"An API is a promise. REST is how you keep it consistently."*

---

## What Is a REST API?

### Conventions for how clients talk to your server over HTTP

Two constraints that matter most:

**Stateless** — each request contains everything the server needs. No session memory between requests.

**Uniform Interface** — resources identified by URLs, interacted with via HTTP methods:

| Method          | Action | Example            |
| --------------- | ------ | ------------------ |
| `GET`           | Read   | `GET /users`       |
| `POST`          | Create | `POST /users`      |
| `PUT` / `PATCH` | Update | `PUT /users/42`    |
| `DELETE`        | Delete | `DELETE /users/42` |

> REST is a convention, not a specification.

---

## Resources and URLs

### Think in nouns, not verbs

**Wrong — verby URLs:**
```
GET  /getUsers
POST /createUser
GET  /deleteUser?id=42
```

**Right — noun-based, method-driven:**
```
GET    /users           list all users
POST   /users           create a new user
GET    /users/:id       get one user
PUT    /users/:id       replace a user
DELETE /users/:id       delete a user
```

**Rule:** URLs are nouns. HTTP methods are verbs.

---

## HTTP Status Codes

### Your API's response language

| Code  | Meaning      | When to use                   |
| ----- | ------------ | ----------------------------- |
| `200` | OK           | Successful GET, PUT, PATCH    |
| `201` | Created      | Successful POST               |
| `204` | No Content   | Successful DELETE             |
| `400` | Bad Request  | Invalid input from client     |
| `401` | Unauthorized | Not authenticated             |
| `403` | Forbidden    | Authenticated but not allowed |
| `404` | Not Found    | Resource doesn't exist        |
| `409` | Conflict     | Duplicate resource            |
| `500` | Server Error | Something broke on the server |

Don't return `200` for everything. Status codes are part of the contract.

---

## Express: Route Parameters and Query Strings

### `req.params` — named segments in the URL path

```javascript
// GET /users/42
app.get('/users/:id', (req, res) => {
  console.log(req.params.id); // "42" (always a string)
});
```

### `req.query` — values after the `?`

```javascript
// GET /users?role=admin&limit=10
app.get('/users', (req, res) => {
  console.log(req.query.role);  // "admin"
  console.log(req.query.limit); // "10" (always a string)
});
```

Always treat query params as optional — the route should work without them.

---

## Express: Request Body

### `req.body` — for POST, PUT, PATCH requests

Requires middleware — add this once, near the top of `server.js`:

```javascript
app.use(express.json());
```
---

### Then in any route:

```javascript
// POST /users
// Request body: 
//   { "name": "Alice", "email": "alice@example.com" }
app.post('/users', (req, res) => {
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json(
	    { error: 'Name and email are required' });
  }

  res.status(201).json({ name, email });
});
```

---

## Express Router

> Don't put everything in server.js

### Router file — `routes/users.js`:
```javascript
const express = require('express');
const router = express.Router();

router.get('/', getAllUsers);
router.post('/', createUser);
router.get('/:id', getUserById);
router.put('/:id', updateUser);
router.delete('/:id', deleteUser);

module.exports = router;
```

---

### server.js — stays clean:

```javascript
const usersRouter = require('./routes/users');
const postsRouter = require('./routes/posts');

app.use('/users', usersRouter);
app.use('/posts', postsRouter);
```

---

## Middleware: The Pipeline

### Any function that runs between request and response

```javascript
// A middleware function signature
(req, res, next) => {
  // do something with req or res
  next(); // pass control to the next handler
}
```
---

### Middleware runs in the order it is declared:

```javascript
// 1. parse JSON body
app.use(express.json());       
// 2. log every request
app.use(logRequests);          
// 3. auth routes under /api
app.use('/api', authenticate); 
// 4. route to handler
app.use('/users', usersRouter);
// 5. catch errors — always last
app.use(errorHandler); 
```

**Types:** built-in (`express.json`), third-party (`morgan`, `cors`), custom (anything you write).

---

## Writing Custom Middleware

### Two patterns you'll use constantly:
#### Logger
#### Error Handler

---

### Request logger:
```javascript
function logger(req, res, next) {
  console.log(`${req.method} ${req.url}`);
  next(); // always call next() or the request hangs
}
```

---

### Error handler — four arguments, always registered last:
```javascript
function errorHandler(err, req, res, next) {
  console.error(err.stack);
  res.status(err.status || 500).json({
    error: err.message || 'Internal server error'
  });
}

app.use(errorHandler); // must be last
```

If you forget `next()` in a regular middleware, the request hangs with no response.

---

## MongoDB and Mongoose

### Why MongoDB, and what Mongoose adds

**MongoDB:**
- Document database — stores JSON-like documents, not rows and tables
- Flexible schema — documents in the same collection can differ
- Natural fit for JavaScript — data is already in JSON format

**Mongoose:**
- ODM (Object Document Mapper) for MongoDB in Node
- Adds **schemas**, **validation**, and **query helpers**
- Models feel like classes with built-in database operations

```bash
npm install mongoose
```

Without Mongoose: raw queries, no validation, no structure.
With Mongoose: define a schema once, get all three for free.

---

## Connecting to MongoDB

```javascript
// db.js
const mongoose = require('mongoose');

async function connectDB() {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log('MongoDB connected');
  } catch (err) {
    console.error('Connection failed:', err.message);
    // crash on startup if DB is unavailable
    process.exit(1); 
  }
}

module.exports = connectDB;
```

---

### Wire up server.js

```javascript
// server.js
require('dotenv').config();
const connectDB = require('./db');

connectDB(); // connect before handling requests
```

### Use a .env file for connect string
and other sensitive bits

```bash
# .env
MONGO_URI=mongodb://localhost:27017/myapp
```

---

## Defining a Mongoose Schema

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
    },
    
```

---

```javascript
    
    email: {
      type: String,
      required: [true, 'Email is required'],
      unique: true,
      lowercase: true,
    },
    role: {
      type: String,
      enum: ['user', 'admin'],
      default: 'user',
    },
  },
  { timestamps: true } // adds createdAt and updatedAt
);
```

---

## Creating the Model

### Schema defines the shape. Model provides the query API.

```javascript
// models/User.js (continued)
const User = mongoose.model('User', userSchema);
module.exports = User;
```

---

```javascript
// Anywhere in your routes
const User = require('../models/User');

// CREATE
const user = await User.create(
	{ name: 'Alice', email: 'alice@example.com' });

// READ
const users = await User.find({ role: 'admin' });
const user  = await User.findById(id);

// UPDATE
const updated = await User.findByIdAndUpdate(id, 
	{ name: 'Alice S.' },
	{ new: true, runValidators: true });

// DELETE
await User.findByIdAndDelete(id);
```

---

## Route: GET and POST

### The consistent pattern for every route handler

```javascript
// GET /users
router.get('/', async (req, res, next) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (err) {
    next(err);
  }
});
```

---

```javascript
// POST /users
router.post('/', async (req, res, next) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    next(err); // ValidationError → errorHandler
  }
});
```

`try/catch` + `next(err)` on every async handler — no exceptions.

---

## Route: GET by ID

```javascript
// GET /users/:id
router.get('/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);
    if (!user) return res.status(404).json({ error: 'User not found' });
    res.json(user);
  } catch (err) {
    next(err); // CastError (bad ID) → errorHandler
  }
});
```

---

## Route: DELETE

```javascript
// DELETE /users/:id
router.delete('/:id', async (req, res, next) => {
  try {
    const user = await User
	    .findByIdAndDelete(req.params.id);
    if (!user) return res.status(404)
	    .json({ error: 'User not found' });
    res.status(204).send();
  } catch (err) {
    next(err);
  }
});
```

---

## Central Error Handler

### Handle Mongoose errors in one place

```javascript
// middleware/errorHandler.js
function errorHandler(err, req, res, next) {
  // Mongoose validation failed
  if (err.name === 'ValidationError') {
    const msgs = Object.values(err.errors)
	    .map(e => e.message);
    return res.status(400)
	    .json({ error: msgs.join(', ') });
  }  
```

---
  
```javascript
  // Duplicate unique field
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    return res.status(409)
	    .json({ error: `Duplicate ${field}` });
  }
  // Bad MongoDB ObjectId
  if (err.name === 'CastError') {
    return res.status(400)
		.json({ error: 'Invalid ID format' });
  }
  res.status(500)
		.json({ error: 'Internal server error' });
}
```

---

## Project Structure

```
my-api/
├── server.js           ← entry point
├── db.js               ← database connection
├── .env                ← gitignored
├── .env.example        ← committed
├── package.json
│
├── models/
│   └── User.js         ← schema + model
│
├── routes/
│   └── users.js        ← Express router for /users
│
└── middleware/
    └── errorHandler.js ← central error handling
```
---

## Testing Your API

### Three tools
- Command-Line (`curl`)
- GUI Application (Postman, Bruno)
- API (jest+supertest)

---

### curl — quick, scriptable:
```bash
curl http://localhost:3000/users

curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'
```
---

### Postman / Bruno — GUI for exploring APIs during development
![Screenshot - Bruno POST Test.jpg](../_assets/images/Screenshot%20-%20Bruno%20POST%20Test.jpg)

---

### Jest + Supertest — automated tests that GitHub Actions will run:
```javascript
test('GET /users returns 200', async () => {
  const res = await request(app).get('/users');
  expect(res.statusCode).toBe(200);
});
```

---

## Summary

### What we covered

- **REST** — nouns in URLs, HTTP methods as verbs, status codes that mean something
- **`req.params`** — path segments; **`req.query`** — after `?`; **`req.body`** — request payload
- **Middleware** — a pipeline; order matters; always call `next()`
- **`express.Router()`** — keeps routes organized, server.js clean
- **MongoDB** — document database; flexible, JSON-native
- **Mongoose** — schemas with validation; `create`, `find`, `findById`, `findByIdAndUpdate`, `findByIdAndDelete`
- **Error handling** — `try/catch` + `next(err)` + one central error handler

**Next up:** Build a complete REST API with MongoDB

---

## Speaker Notes

### Slide 2 (REST)
REST is commonly confused with "any JSON API." Clarify: it's a set of conventions, not a technology. The stateless constraint is the most important one — it's why you don't use server-side sessions and why JWTs exist.

### Slide 4 (Status Codes)
Quick poll: "Who's ever gotten a 200 response that was actually an error?" Almost everyone has. That's the most common REST anti-pattern. Status codes are part of the API contract.

### Slide 8–9 (Middleware)
The "runs in order" rule is what students get wrong. Draw the pipeline on the board. Ask: "What happens if you forget `next()`?" (Request hangs.) "What if you call `next()` after `res.json()`?" (Headers already sent error.) These are the two middleware bugs they'll hit in lab.

### Slides 14–15 (Routes)
Walk through the pattern slowly: `async`, `try/catch`, check for null before returning, `next(err)` in every catch. Once students internalize this pattern, writing new routes is mechanical. That's the goal.

### Slide 18 (Testing)
Mention Postman for the lab — students will want a GUI. Point to Supertest as the path that connects to CI/CD. "The test you write today is what GitHub Actions will run."
