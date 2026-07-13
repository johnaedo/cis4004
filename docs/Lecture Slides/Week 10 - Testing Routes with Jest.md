---
share_cis4004: "true"
share_cop4331c: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---

# Testing Express Routes with Jest
### A beginner's guide using a real project

---

## What are we doing and why?

When you write a route like `POST /api/users/login`, how do you know it actually works?

You could open Postman and test it manually every time. But that gets tedious fast — and it doesn't scale when your app has dozens of routes.

**Automated tests** solve this. You write code that *calls* your routes and *checks* the responses. Run them any time with one command. If something breaks, you know immediately.

In this tutorial we'll write tests for the four routes in `routes/users.js`:

- `POST /api/users/register`
- `POST /api/users/login`
- `GET  /api/users/profile`
- `PUT  /api/users/profile`

---

## Tools we're using

**Jest** — the test runner. It finds your test files, runs them, and tells you what passed and failed.

**Supertest** — lets you make fake HTTP requests to your Express app without starting a real server. Instead of `http://localhost:3001/api/users/login` you just write `request(app).post("/api/users/login")`.

Install both with:

```bash
npm install --save-dev jest supertest
```

You'll also need Babel so Jest can understand your ES Module syntax 
(`import`/`export`):

```bash
npm install --save-dev babel-jest \
@babel/core @babel/preset-env \
babel-plugin-transform-import-meta
```

---

## Why don't we just use a real database?

You might be thinking: *"why not just connect to the database and test for real?"*

Three reasons:

1. **Speed** — a real DB query takes milliseconds. Multiply that by hundreds of tests and your test suite takes minutes. Mock queries are instant.

2. **Isolation** — tests that write to a real DB leave data behind. That data affects other tests in unpredictable ways. Mocks start clean every time.

3. **Control** — to test what happens when the DB crashes, you need to *make* the DB crash. You can't do that reliably with a real connection. With mocks, you just tell the mock to throw an error.

The word for this technique is **mocking** — replacing a real dependency with a fake one that you control.

---

## What are we mocking?

Our `users.js` controller imports three things that touch the outside world:

```js
// MySQL database
import { pool }  from "../config/database.js";
// password hashing 
import bcrypt    from "bcrypt";
// token signing                 
import jwt       from "jsonwebtoken";           
```

We'll replace all three with fakes. The fakes behave exactly like the real things — they just don't touch a database, don't do real hashing, and don't generate real tokens.

---

## Setting up the mocks

At the very top of your test file, before any `import` statements, declare your mocks:
<style>
.reveal pre {
	font-size: 0.8em;
}
</style>
```js
jest.mock("../config/database.js", () => ({
  pool: { query: jest.fn() },
}));

jest.mock("bcrypt", () => ({
  genSalt: jest.fn().mockResolvedValue("salt"),
  hash:    jest.fn().mockResolvedValue("hashed_password"),
  compare: jest.fn(),
}));

jest.mock("jsonwebtoken", () => ({
  sign:   jest.fn(() => "mocked.jwt.token"),
  verify: jest.fn(),
}));

jest.mock("../config/dotenv.js", () => {});
```
---


`jest.fn()` creates a fake function. `mockResolvedValue(x)` makes it return a resolved Promise with value `x` — useful for async functions like `bcrypt.genSalt`.

> **Why before imports?** Jest hoists `jest.mock()` calls to the top of the file automatically, but declaring them first makes this explicit and avoids confusion.

---

## Building a test app

Rather than starting the full `server.js`, we build a minimal Express app that mounts only the users router:

```js
function buildApp() {
  const app = express();
  app.use(express.json());
  app.use("/api/users", require("../routes/users.js").default);
  app.use(require("../middleware/errorHandler.js").errorHandler);
  return app;
}
```

This keeps tests focused. We're testing the users routes — not CORS, not the budgets router, not anything else.

> [!IMPORTANT]
> We use `require()` here instead of `import` because Jest runs in CommonJS mode under the hood. `mod.default` unwraps the ES Module default export after Babel compiles it.

---

## A fake user to test with

Define a realistic mock database row near the top of the file. This is what MySQL would actually return:

```js
const mockUser = {
  id: 1,
  username: "alice",
  email: "alice@example.com",
  password_hash: "$2b$10$mockedhashvalue",
  created_at: "2024-01-01T00:00:00.000Z",
};
```

We reuse this object across many tests. When a test needs the database to return a user, it tells `pool.query` to return this object.

---

## MySQL2 returns nested arrays — this trips everyone up

The `pg` library (Postgres) returns `{ rows: [...] }`.

The `mysql2` library (what this project uses) returns `[[rows], [fields]]` — an array of arrays.

This is why the controller destructures like this:

```js
const [results] = await pool.query(query, params);
// results is now the array of rows
```

Your mocks must match this shape. A SELECT that returns one user looks like:

```js
pool.query.mockResolvedValueOnce([[mockUser]]);
//                               ^^ outer array = [rows, fields]
//                                ^^ inner array = the rows
```

Get this wrong and the controller silently receives `undefined` instead of your data.

---

## Your first test — successful login

Here's a complete test for a successful login with an email:

```js
it("returns 200 with user and token when logging in with a valid email", async () => {
  pool.query.mockResolvedValueOnce([[mockUser]]); // DB returns the user
  bcrypt.compare.mockResolvedValueOnce(true);     // password matches

  const res = await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice@example.com", password: "password123" });

  expect(res.status).toBe(200);
  expect(res.body.token).toBe("mocked.jwt.token");
  expect(res.body.user).toMatchObject({
    id: 1,
    username: "alice",
    email: "alice@example.com",
  });
});
```

Read it like a story:
1. *When* the DB returns Alice's record...
2. *And* the password check passes...
3. *Then* a POST to `/login` should return 200, a token, and the user object.

---

## `mockResolvedValueOnce` vs `mockReturnValue`

You'll see `mockResolvedValueOnce` used everywhere. The `Once` part is important.

| Method | Behaviour |
|---|---|
| `mockReturnValue(x)` | Returns `x` for every call, forever |
| `mockResolvedValueOnce(x)` | Returns `x` for the *next* call only, then resets |

Using `Once` means each test sets up exactly the responses it expects. If a test accidentally calls the DB twice, the second call returns `undefined` and the test fails loudly — instead of silently reusing a value from a previous test.

---

## Testing failure cases

Testing success is easy. Testing failures is just as important — often more so.

```js
it("returns 401 when the password is wrong", async () => {
  pool.query.mockResolvedValueOnce([[mockUser]]); // user exists
  bcrypt.compare.mockResolvedValueOnce(false);    // but password fails

  const res = await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice@example.com", password: "wrongpassword" });

  expect(res.status).toBe(401);
  expect(res.body.error).toBe("Invalid credentials");
});

it("returns 401 when the user does not exist", async () => {
  pool.query.mockResolvedValueOnce([[]]); // empty results — no user found

  const res = await request(app)
    .post("/api/users/login")
    .send({ identifier: "nobody@example.com", password: "password123" });

  expect(res.status).toBe(401);
  expect(bcrypt.compare).not.toHaveBeenCalled(); // should never reach password check
});
```
---

> That last assertion — `not.toHaveBeenCalled()` — is checking a **security property**. If the user doesn't exist, we should never attempt a password comparison. This prevents timing attacks.

---

## Testing that the right DB query was made

Sometimes you want to verify not just *what* came back, but *what was sent* to the database.

The `loginUser` controller uses a regex to decide whether `identifier` is an email or a username. Let's test that logic directly.

---
```js
it("queries by email when identifier looks like an email", async () => {
  pool.query.mockResolvedValueOnce([[mockUser]]);
  bcrypt.compare.mockResolvedValueOnce(true);

  await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice@example.com", password: "password123" });


  expect(pool.query).toHaveBeenCalledWith(
    "SELECT * FROM users WHERE email = ?",
    ["alice@example.com"],
  );
});
```
---

```js
it("queries by username when identifier does not look like an email", async () => {
  pool.query.mockResolvedValueOnce([[mockUser]]);
  bcrypt.compare.mockResolvedValueOnce(true);

  await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice", password: "password123" });

  expect(pool.query).toHaveBeenCalledWith(
    "SELECT * FROM users WHERE username = ?",
    ["alice"],
  );
});
```

---

## Testing database errors

What happens if the database goes down mid-request? The controller should handle it gracefully and return a 500.

```js
it("returns 500 when the database throws", async () => {
  jest.spyOn(console, "error").mockImplementation(() => {});
  pool.query.mockRejectedValueOnce(new Error("Connection lost"));

  const res = await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice@example.com", password: "password123" });

  expect(res.status).toBe(500);
  console.error.mockRestore();
});
```
---

`mockRejectedValueOnce` makes the mock throw a rejected Promise — simulating a real database failure.

We suppress `console.error` here with `mockImplementation(() => {})` because the controller calls `console.error` when it catches the error. Without this, the error message prints to the test output and looks like a failure even when the test passes. `mockRestore()` puts `console.error` back to normal afterward.

---

## Protected routes — how auth works here

The `GET /profile` and `PUT /profile` routes are protected by `auth.js` middleware:

```js
router.get("/profile", auth, UsersController.getUserProfile);
router.put("/profile", auth, UsersController.updateProfile);
```

Here's the key thing about how `auth.js` works in this project:

```js
// auth.js — simplified
const decoded = jwt.verify(token, process.env.JWT_SECRET);
const [rows]  = await pool.query("SELECT ... WHERE id = ?", [decoded.id]);
req.user = rows[0];
next();
```

It calls **both** `jwt.verify` **and** `pool.query`. That means every test for a protected route needs to mock both of these — before the controller's own queries run.

---

## The `mockAuthenticatedUser` helper

Because every protected route test needs the same setup, we extract it into a helper:

```js
function mockAuthenticatedUser(user = mockUser) {
  jwt.verify.mockReturnValueOnce({ id: user.id }); // JWT decodes successfully
  pool.query.mockResolvedValueOnce([[user]]);        // auth middleware DB lookup
}
```

Call this once at the start of any test that hits a protected route. Think of it as *"log Alice in"* — after this, `req.user` will be set to `mockUser` when the controller runs.

---

## Testing a protected route

Now a test for `GET /profile` is clean and readable:

```js
it("returns the user profile when authenticated", async () => {
  mockAuthenticatedUser();                            // sets up auth
  pool.query.mockResolvedValueOnce([[mockUser]]);     // controller's own query

  const res = await request(app)
    .get("/api/users/profile")
    .set("Authorization", "Bearer valid.jwt.token"); // send a token header

  expect(res.status).toBe(200);
  expect(res.body).toMatchObject({
    id: 1,
    username: "alice",
    email: "alice@example.com",
  });
});
```
---

Notice we're calling `pool.query` mock twice:
1. Once inside `mockAuthenticatedUser()` — for the auth middleware
2. Once after — for the controller itself

The order matters. Jest's mock queue is first-in, first-out.

---

## `toMatchObject` vs `toEqual`

You'll see both used in the tests. They mean slightly different things:

```js
// toEqual — exact match, every field must be present
expect(res.body).toEqual({ id: 1, username: "alice", email: "alice@example.com" });

// toMatchObject — partial match, only the listed fields are checked
expect(res.body).toMatchObject({ id: 1, username: "alice" });
```

Use `toMatchObject` when the response might contain extra fields you don't care about (like `created_at`). Use `toEqual` when you want to assert the *exact* shape of the response.

---

## Testing a security property — no password_hash in responses

The database row contains `password_hash`. The response must never include it.

```js
it("does not expose password_hash in the response", async () => {
  pool.query.mockResolvedValueOnce([[mockUser]]);
  bcrypt.compare.mockResolvedValueOnce(true);

  const res = await request(app)
    .post("/api/users/login")
    .send({ identifier: "alice@example.com", password: "password123" });

  expect(res.body.user.password_hash).toBeUndefined();
});
```

This kind of test might seem trivial, but it catches a real class of bug. If someone changes the controller and accidentally sends the whole user row instead of a picked subset, this test fails immediately.

---

## `afterEach` — keeping tests independent

Add this inside every `describe` block:

```js
afterEach(() => jest.clearAllMocks());
```

This resets all mock call counts and return values after each test. Without it:

- A mock set up for test 1 might still be active during test 2
- `toHaveBeenCalledWith` assertions might count calls from previous tests
- Tests start depending on each other's side effects

Each test should be a clean slate. `clearAllMocks` enforces that.

---

## A note on `auth.js` — a gap worth knowing

- There is an important behaviour to understand about the auth middleware in this project. When a request arrives with no token, or a bad token, `auth.js` does **not** return a 401. It sets `req.user = null` and calls `next()`.

- This means the request reaches the controller with `req.user = null`. When the controller then tries to read `req.user.id`, it crashes with a null reference error and returns 500.

---

This test documents that behavior:

```js
it("does not block the request when no token is provided", async () => {
  jest.spyOn(console, "error").mockImplementation(() => {});

  const res = await request(app).get("/api/users/profile");

  expect(res.status).not.toBe(401); // auth never returns 401 — this is the gap
  console.error.mockRestore();
});
```

The fix would be to add a guard at the top of each protected controller method:

```js
if (!req.user) return res.status(401).json({ error: "Unauthorized" });
```

---

## Putting it all together — full test structure

Here is the skeleton of the complete test file so you can see how everything fits:

```
users.test.js
│
├── jest.mock() calls         ← mocks declared before imports
├── imports                   ← supertest, express, mocked modules
├── buildApp()                ← minimal test Express app
├── mockUser                  ← fake DB row used across tests
├── mockAuthenticatedUser()   ← helper for protected routes
│
```

---
```
├── describe: POST /register
│   ├── returns 201 on success
│   ├── hashes the password
│   ├── returns 400 on duplicate entry
│   └── returns 500 on DB error
│
├── describe: POST /login
│   ├── success with email
│   ├── success with username
│   ├── queries by email vs username
│   ├── 401 when user not found
│   ├── 401 when password wrong
│   ├── 500 on DB error
│   ├── signs JWT with user id
│   └── no password_hash in response
│
```
---
```
├── describe: GET /profile
│   ├── returns profile when authenticated
│   ├── no password_hash in response
│   ├── 500 on DB error
│   └── documents non-blocking auth behaviour
│
└── describe: PUT /profile
    ├── returns updated profile on success
    ├── 401 when current password wrong
    ├── 404 when user not found
    ├── 400 on duplicate username/email
    ├── 500 on DB error
    └── UPDATE query never touches password_hash
```

---

## Running the tests

From your `server/` directory:

```bash
npx jest users.test.js
```

To run all test files at once:

```bash
npx jest
```

To run in watch mode (re-runs on every file save — useful while writing tests):

```bash
npx jest --watch
```

---

A passing run looks like this:

```
PASS  users.test.js
  POST /api/users/register
    ✓ returns 201 with user and token on success
    ✓ hashes the password before storing it
    ✓ returns 400 when username or email is already taken
    ✓ returns 500 when the database throws an unexpected error
  POST /api/users/login
    ✓ returns 200 with user and token when logging in with a valid email
    ...

Tests: 20 passed, 20 total
```

---

## What we covered

- Why automated tests are better than manual Postman testing
- What mocking is and why we mock the database, bcrypt, and JWT
- How mysql2's nested array return shape affects your mocks
- How to test success cases, failure cases, and security properties
- How the auth middleware works and what its current limitation is
- How to keep tests isolated with `afterEach` and `mockResolvedValueOnce`

## What comes next

- Writing tests for the other routers: `categories`, `transactions`, `budgets`
- Testing routes that require query parameters and URL parameters
- Setting up a test database for true end-to-end integration tests
- Measuring code coverage with `npx jest --coverage`
