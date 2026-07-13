---
share_cis4004: "true"
share_cop4331c: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---

# Authentication and Authorization with JWT
### Knowing who's asking, and what they're allowed to do

*"Security is not a feature you add at the end. It's a constraint you design around from the start."*

---

## Two Concepts, Often Confused

### Authentication vs Authorization

| Concept            | Question It Answers           | Example                             |
| ------------------ | ----------------------------- | ----------------------------------- |
| **Authentication** | *Who are you?*                | Verifying a username and password   |
| **Authorization**  | *What are you allowed to do?* | Checking if a user has admin rights |

**The order matters: you must authenticate before you can authorize.**

A request can fail at either step:
- **401 Unauthorized** — not authenticated (we don't know who you are)
- **403 Forbidden** — authenticated but not authorized (we know who you are, but you can't do this)

> Think of a building with a keycard lock. Authentication is swiping your card to prove you work there. Authorization is whether your card opens the server room specifically.

---

## The Problem with Sessions

### Why not just use server-side sessions?

Traditional session-based auth:
```
1. User logs in → server creates a session, 
   stores it in memory (or DB)
2. Server sends back a session ID in a cookie
3. Every request includes the cookie
4. Server looks up the session ID to identify the user
```

This works. But:
- **Stateful** — the server must store and look up session data for every request
- **Scaling problem** — if you add a second server, it doesn't have the first server's sessions
- **Session store required** — Redis or DB needed to share sessions across instances
- **Mobile/API unfriendly** — cookies are a browser concept; mobile apps and other APIs don't use them naturally

**JWTs are a stateless alternative.**

---

## What Is a JWT?

### JSON Web Token — a self-contained credential

A JWT is a string that encodes identity and claims, signed by the server.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
...many more characters follow...
```

Three parts, separated by dots:

| Part          | Content                                        | Encoded as |
| ------------- | ---------------------------------------------- | ---------- |
| **Header**    | Algorithm and token type                       | Base64URL  |
| **Payload**   | Claims (userId, role, expiry)                  | Base64URL  |
| **Signature** | HMAC of header + payload, signed with a secret | Base64URL  |

**Base64URL is not encryption.** Anyone can decode the header and payload. The signature is what makes the token trustworthy — it proves the token was issued by your server and hasn't been tampered with.

---

## JWT Anatomy: Decoded

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (claims):**
```json
{
  "userId": "64abc123",
  "email": "alice@example.com",
  "role": "admin",
  "iat": 1700000000,
  "exp": 1700086400
}
```

---
## Claims

**Standard claim names:**
- `iat` — issued at (Unix timestamp)
- `exp` — expiration time (Unix timestamp)
- `sub` — subject (usually the user ID)
- `iss` — issuer

> **Never put sensitive data in a JWT payload.** Passwords, credit card numbers, secrets — anyone who intercepts the token can read the payload. Put only what downstream services need to identify and authorize the user.

---

## How JWT Auth Works - the Full Cycle

<style>
.reveal pre {
	font-size: 0.8em;
}
</style>
```
┌──────────┐                         ┌──────────────┐
│  Client  │                         │    Server    │
└────┬─────┘                         └──────┬───────┘
     │                                      │
     │  POST /auth/login                    │
     │  { email, password }                 │
     │─────────────────────────────────────▶│
     │                                      │ 1. Find user by email
     │                                      │ 2. Compare password hash
     │                                      │ 3. Sign JWT with secret
     │  200 { token: "eyJ..." }             │
     │◀─────────────────────────────────────│
     │                                      │
     │  GET /api/profile                    │
     │  Authorization: Bearer eyJ...        │
     │─────────────────────────────────────▶│
     │                                      │ 4. Verify JWT signature
     │                                      │ 5. Check expiry
     │                                      │ 6. Attach user to req
     │  200 { user data }                   │
     │◀─────────────────────────────────────│
```

**The server never stores the token.** Validity is proven by the signature alone.

---

## Storing Passwords: bcrypt

### Never store plaintext passwords

**What not to do:**
```javascript
// NEVER DO THIS
user.password = req.body.password;
await user.save();
```

If your database is breached, every user's password is exposed — and since people reuse passwords, it compromises their accounts everywhere.

---

**What to do: hash with bcrypt**

bcrypt is a slow, one-way hashing algorithm designed for passwords:
- **One-way:** you cannot reverse a bcrypt hash to get the original password
- **Salted:** each hash includes a random salt, so identical passwords produce different hashes
- **Intentionally slow:** configurable work factor (cost) makes brute-force attacks expensive

```bash
npm install bcrypt
```

```javascript
const bcrypt = require('bcrypt');
const SALT_ROUNDS = 12;  // work factor — higher is slower/safer

// Hashing (on registration)
const hash = await bcrypt.hash(plainTextPassword, SALT_ROUNDS);

// Comparing (on login)
const isMatch = await bcrypt.compare(plainTextPassword, storedHash);
// → true or false
```

---

## The User Model with Password Hashing

### Hashing at the model layer with a Mongoose hook

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');

const userSchema = new mongoose.Schema({
  name:     { type: String, required: true, trim: true },
  email:    { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true, minlength: 8 },
  role:     { type: String, enum: ['user', 'admin'], default: 'user' },
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();  // only hash if changed
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

```
---
## Safely return user object

```js
// Never return the password field in responses
userSchema.set('toJSON', {
  transform: (doc, ret) => {
    delete ret.password;
    delete ret.__v;
    return ret;
  },
});

// Instance method to compare passwords
userSchema.methods.comparePassword = async function (candidate) {
  return bcrypt.compare(candidate, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

---

## The Auth Routes: Register and Login

### Two endpoints, two flows

```javascript
// routes/auth.js
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');
const router = express.Router();

// POST /auth/register
router.post('/register', async (req, res, next) => {
  try {
    const { name, email, password } = req.body;
    const user = await User.create({ name, email, password });
    // pre('save') hook hashes the password automatically
    res.status(201).json({ message: 'Account created', user });
  } catch (err) {
    next(err);
  }
});

```
---
```js
// POST /auth/login
router.post('/login', async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email }).select('+password');
    if (!user) return res.status(401).json({ error: 'Invalid credentials' });

    const isMatch = await user.comparePassword(password);
    if (!isMatch) return res.status(401).json({ error: 'Invalid credentials' });

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({ token });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```
---

> **Always return the same error message for bad email and bad password.** Separate messages (`"user not found"` vs `"wrong password"`) let attackers enumerate valid email addresses.

---

#### The Authenticate Middleware
```javascript
// middleware/authenticate.js
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;  // { userId, role, iat, exp }
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}

module.exports = authenticate;
```

---

## Usage — protect any route:
```javascript
const authenticate = require('../middleware/authenticate');

// Protect an entire router
app.use('/api', authenticate);

// Protect individual routes
router.get('/profile', authenticate, getProfile);
router.delete('/:id', authenticate, deleteUser);
```

---

## Authorization: Role-Based Access Control

### Authentication proves identity. Authorization enforces permissions.

```javascript
// middleware/authorize.js
function authorize(...roles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

module.exports = authorize;
```

---

## Usage
```javascript
const authenticate = require('../middleware/authenticate');
const authorize = require('../middleware/authorize');

// Any authenticated user
router.get('/profile', authenticate, getProfile);

// Admin only
router.delete('/users/:id', authenticate, authorize('admin'), deleteUser);

// Admin or moderator
router.patch('/posts/:id/flag', authenticate, authorize('admin',
	'moderator'), flagPost);
```

**The middleware chain:** `authenticate` runs first (proves identity, attaches `req.user`), then `authorize` checks the role.

---

## Token Expiry and Refresh Tokens

### Short-lived access tokens + long-lived refresh tokens

**The problem with long-lived tokens:**
- If a token is stolen, the attacker has access until it expires
- You can't easily invalidate a JWT (the server is stateless)

**The solution: two-token pattern**

| Token             | Lifespan        | Stored                            | Purpose                             |
| ----------------- | --------------- | --------------------------------- | ----------------------------------- |
| **Access token**  | 15 min – 1 hour | Memory / JS variable              | Sent with every API request         |
| **Refresh token** | 7–30 days       | HttpOnly cookie or secure storage | Used only to get a new access token |

---
```javascript
// Login: issue both tokens
const accessToken = jwt.sign(
  { userId: user._id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '15m' }
);

const refreshToken = jwt.sign(
  { userId: user._id },
  process.env.JWT_REFRESH_SECRET,
  { expiresIn: '7d' }
);

res.json({ accessToken, refreshToken });
```

---

```javascript
// POST /auth/refresh — get a new access token
router.post('/refresh', (req, res) => {
  const { refreshToken } = req.body;
  try {
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    const accessToken = jwt.sign(
      { userId: decoded.userId },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );
    res.json({ accessToken });
  } catch (err) {
    res.status(401).json({ error: 'Invalid or expired refresh token' });
  }
});
```

---

## Security: What Can Go Wrong

### The most common JWT implementation mistakes

**1. Weak or missing secret**
```bash
# Bad
JWT_SECRET=secret
JWT_SECRET=password123

# Good — generate a strong random secret
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

**2. Sensitive data in the payload**
```javascript
// Bad — never put this in a JWT
{ password: "plaintext", creditCard: "4111..." }

// Good — only identifiers and public claims
{ userId: "64abc123", role: "user" }
```

---

**3. Not checking expiry**
`jwt.verify()` checks expiry automatically — but only if you use it. Never decode without verifying.

**4. Storing access tokens in localStorage**
localStorage is accessible to any JavaScript on the page — including injected scripts (XSS). Prefer memory storage for access tokens, HttpOnly cookies for refresh tokens.

**5. Not rotating refresh tokens**
After a refresh token is used to get a new access token, invalidate it and issue a new one. If a refresh token is stolen and used, you'll detect it when the legitimate user's copy is rejected.

---

## Secrets that must never be committed

```bash
# .env
JWT_SECRET=a64-character-random-hex-string-generated-with-crypto
JWT_REFRESH_SECRET=another-different-64-character-random-hex-string
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

```bash
# .env.example
JWT_SECRET=replace_with_output_of_node_crypto_randomBytes_64_hex
JWT_REFRESH_SECRET=replace_with_a_different_random_secret
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
```

```javascript
// Generate a secret — run this once in your terminal
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

> Using the same secret for access and refresh tokens defeats the purpose of having two. If one secret is compromised, both token types are compromised. Always use separate secrets.

---

## Updated Project Structure

### Where auth files live

```
my-api/
├── server.js
├── db.js
├── .env
├── .env.example
│
├── models/
│   └── User.js           ← password hashing hook, comparePassword method
│
├── routes/
│   ├── auth.js           ← POST /auth/register, POST /auth/login, POST /auth/refresh
│   └── users.js          ← protected routes
│
└── middleware/
    ├── authenticate.js   ← verify JWT, attach req.user
    ├── authorize.js      ← check role
    └── errorHandler.js
```

---

## In server.js
```javascript
app.use('/auth', require('./routes/auth'));
// all user routes protected
app.use('/users', authenticate, require('./routes/users'));  
```

---

## What Is OAuth 2.0?


- **The problem OAuth solves:**
Users don't want to create yet another account. Developers don't want to store passwords. Both want to use existing identity providers (Google, GitHub, etc.).

- **OAuth 2.0** is an authorization framework that lets a user grant your app limited access to their account at another service — without giving you their password.

---

```
User clicks "Sign in with Google"
        │
        ▼
Redirected to Google's login page
        │
        ▼
User authenticates with Google (not your app)
        │
        ▼
Google redirects back to your app with an authorization code
        │
        ▼
Your server exchanges the code for an access token (server-to-server)
        │
        ▼
Your server calls Google's API to get the user's profile
        │
        ▼
Your server creates/finds the user in your DB, issues your own JWT
```

> Your app never sees the user's Google password. Google handles authentication; your app handles authorization from that point on.

---

## OAuth Key Concepts


| Term                     | Meaning                                                                   |
| ------------------------ | ------------------------------------------------------------------------- |
| **Resource Owner**       | The user                                                                  |
| **Client**               | Your application                                                          |
| **Authorization Server** | Google, GitHub, etc. — issues tokens                                      |
| **Resource Server**      | The API you're calling (e.g., Google Profile API)                         |
| **Authorization Code**   | Short-lived code exchanged for tokens (never exposed to browser)          |
| **Access Token**         | Credential for calling the provider's API                                 |
| **Scope**                | What your app is asking permission for (`email`, `profile`, `read:repos`) |

**The authorization code flow** (what "Sign in with Google" uses) is the most secure. The code is exchanged server-to-server — the access token never appears in the browser URL or JavaScript.

---

## OAuth with Passport.js

Passport.js is the standard Node.js library for auth strategies — including OAuth.

```bash
npm install passport passport-google-oauth20
```

---

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy(
  {
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback',
  },
  async (accessToken, refreshToken, profile, done) => {
    // profile contains the user's Google info
    let user = await User.findOne({ googleId: profile.id });

    if (!user) {
      user = await User.create({
        googleId: profile.id,
        name: profile.displayName,
        email: profile.emails[0].value,
      });
    }

    // Issue your own JWT here
    done(null, user);
  }
));
```

---

```javascript
// Routes
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { session: false }),
  (req, res) => {
    const token = jwt.sign(
      { userId: req.user._id, role: req.user.role },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );
    res.json({ token });
  }
);
```

---

## User Model for OAuth

```javascript
const userSchema = new mongoose.Schema({
  name:     { type: String, required: true },
  email:    { type: String, required: true, unique: true },
  role:     { type: String, enum: ['user', 'admin'], default: 'user' },

  // Local auth
  // not required — OAuth users have no password
  password: { type: String, minlength: 8 },  

  // OAuth
  // sparse: unique index ignores null values
  googleId: { type: String, sparse: true },    
  githubId: { type: String, sparse: true },
}, { timestamps: true });

// Only hash password if it exists and was modified
userSchema.pre('save', async function (next) {
  if (!this.password || !this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});
```
---

> `sparse: true` on the index means documents without a `googleId` (local-only users) don't violate the unique constraint. Without it, two local users with no `googleId` would conflict.

---

## Summary

### What we covered

- **Authentication** proves identity; **authorization** enforces permissions — separate concerns, applied in order
- **Sessions** are stateful and hard to scale; **JWTs** are stateless and self-contained
- A JWT has three parts: header, payload, signature — the payload is readable, not encrypted
- **bcrypt** hashes passwords one-way with a salt — never store plaintext
- The Mongoose `pre('save')` hook keeps hashing logic in the model layer
- The **authenticate middleware** verifies the JWT and attaches `req.user` to the request
- The **authorize middleware** checks `req.user.role` against allowed roles
- Short-lived **access tokens** + long-lived **refresh tokens** limit the blast radius of theft
- JWTs have failure modes: weak secrets, sensitive payload data, insecure storage

**Bonus:**
- **OAuth 2.0** delegates authentication to a trusted provider — your app never sees the user's provider password
- **Passport.js** implements OAuth strategies and other auth patterns in Node

---

## Speaker Notes

### Slide 2 (Auth vs Authz)
Ask: "Has anyone ever gotten a 403 from an API they were using? What does that tell you about your account?" The distinction between 401 and 403 is something most students have encountered as API consumers but never thought about as API builders.

### Slide 4–5 (JWT Anatomy)
Go to [jwt.io](https://jwt.io) live. Paste in a token and show the decoded payload. Then modify the payload and show how the signature changes. Then show what happens if you change the payload but keep the old signature — verification fails. This makes the security model concrete.

### Slide 7 (bcrypt)
Show the actual timing difference: `bcrypt.hash('password', 10)` vs `bcrypt.hash('password', 14)`. The difference in milliseconds is the point — it's slow on purpose. Ask: "If an attacker can try 1 billion passwords per second against MD5, how many can they try per second against bcrypt with cost factor 12?" (Answer: a few thousand at most.)

### Slide 9 (Auth Routes)
Emphasize the identical error message for wrong email and wrong password. This is the kind of security detail that's easy to overlook and genuinely matters. Enumerate-the-user-base attacks are real.

### Slide 12 (Refresh Tokens)
This is where complexity spikes. Gauge the room. If students are struggling with the basic JWT flow, skip the refresh token implementation details and just explain the concept. The demo will implement a simpler version with longer-lived access tokens.

### Slide 13 (What Can Go Wrong)
Ask: "Where have you seen 'Sign in with Google' and not thought twice about it? What data is that site getting about you?" This connects OAuth (next slides) to things students already use.

### Slides 16–19 (OAuth — Bonus)
Only cover if you're ahead of time. Don't rush through it. It's better to cut it entirely and refer students to the Passport.js docs than to cover it superficially. The concepts in slides 16–17 are worth explaining even if you skip the code in 18–19.
