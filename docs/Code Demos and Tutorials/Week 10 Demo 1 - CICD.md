---
share_cis4004: "true"
site-folder: docs/Code Demos and Tutorials/
theme: ucf-knights.css
height: "1080"
width: "1920"
---
# Week 10 Demo 1: GitHub Actions

**Duration:** 60 minutes
**Format:** Instructor live-codes; students follow along on their own repos
**Prerequisite:** Lecture 1 (CI/CD), project repo on GitHub with at least one test

---

## Instructor Overview

This lab is structured as a **guided build**, not a lecture. You code, students code. Move at a pace where most of the room is with you — use check-ins frequently. The stretch goal (deploy step) is for fast finishers; don't rush the main flow to reach it.

**End state:** Every student has a working GitHub Actions workflow that:
- Runs lint + tests on every push and pull request
- Blocks PR merges if CI fails
- (Stretch) SSHes into a VM to deploy

---

## Setup Checklist (Before Lab Starts)

Students should have:
- [ ] A Node.js project repo on GitHub (can be their course project)
- [ ] `npm test` running locally and passing
- [ ] ESLint installed OR willing to install it during lab
- [ ] GitHub repo open in browser
- [ ] Code editor open

Instructor should have:
- [ ] A demo repo ready (separate from students' repos)
- [ ] Terminal and editor visible on projector
- [ ] GitHub Actions tab open in browser

---

## Part 1 — Warm-Up and Repo Audit (5 min)

### Instructor Script

> "Before we write a single line of YAML, let's make sure our repos are ready. CI is only as useful as the tests and scripts it runs."

**Ask students to verify — do it yourself live:**

```bash
# 1. Confirm npm test works
npm test

# 2. Confirm what scripts exist
cat package.json | grep '"scripts"' -A 10
```

**What to look for in `package.json`:**
```json
{
  "scripts": {
    "test": "jest",          // or mocha, vitest, etc.
    "lint": "eslint .",      // we'll add this if missing
    "start": "node server.js"
  }
}
```

**If a student has no test script:**
```bash
# Quick fix — install Jest and add a trivial test
npm install --save-dev jest
```

```javascript
// test/smoke.test.js
test('true is true', () => {
  expect(true).toBe(true);
});
```

```json
// package.json
"scripts": {
  "test": "jest"
}
```

> "A CI pipeline that has nothing to run is useless. The test script is the contract CI enforces."

---

## Part 2 — Add ESLint (8 min)

### Why lint in CI?

> "Lint is a fast, cheap check that catches real bugs and enforces style. It costs about 10 seconds in CI and catches things code review misses."

### Live code: Install and configure ESLint

```bash
npm init @eslint/config@latest
```

Follow the prompts (Node.js, CommonJS or ESM, JSON config). Then:

```bash
# Test it runs
npx eslint .

# Add to package.json scripts
```

```json
"scripts": {
  "lint": "eslint .",
  "test": "jest"
}
```

**Introduce a deliberate error:**

```javascript
// In any .js file
const x = 1
const y = 2   // unused variable — ESLint will catch this
```

```bash
npx eslint .
# Should show an error
```

Fix it, confirm it passes:
```bash
npx eslint .   # clean output
```

> "This is what CI will run. If it passes here, it passes there."

**Commit and push:**
```bash
git add .
git commit -m "Add ESLint configuration"
git push
```

---

## Part 3 — Write the CI Workflow (20 min)

### Create the workflow file

```bash
mkdir -p .github/workflows
touch .github/workflows/ci.yml
```

### Build it step by step — type this live, explain each line

```yaml
name: CI
```
> "This is the name shown in the GitHub Actions tab."

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```
> "This runs on every push to main, and on every PR targeting main. These are the two moments when we most need confidence."

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint
```

> "Notice `npm ci` not `npm install`. `ci` is stricter — it fails if `package-lock.json` is out of sync. That's what you want in CI."

```yaml
  test:
    runs-on: ubuntu-latest
    needs: lint

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

> "The `needs: lint` line means tests only run if lint passes. No point running 2 minutes of tests if the code has lint errors. Fail fast."

**Full file for reference:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

**Commit and push:**
```bash
git add .github/
git commit -m "Add CI workflow"
git push
```

### Check it running

> "Go to your repo on GitHub. Click the **Actions** tab. You should see the workflow running right now."

Walk through the UI:
- Workflow run page
- Expanding individual jobs
- Expanding individual steps
- The green checkmark when it passes

**Check-in:** Raise your hand when you see green. (Wait for ~80% before moving on.)

---

## Part 4 — Make CI Fail on Purpose (7 min)

### Why this matters

> "CI that you've never seen fail is CI you don't trust. Let's break it intentionally."

### Break the lint step

```javascript
// Add to any .js file — introduce an unused variable
const badVariable = "this will fail lint"
```

```bash
git add .
git commit -m "Intentional lint error"
git push
```

Go to GitHub → Actions. Watch the `lint` job fail. Notice:
- `test` job never starts (because `needs: lint`)
- Red X on the commit in the commit list

**Fix it and push again:**
```bash
# Remove the bad variable
git add .
git commit -m "Fix lint error"
git push
```

> "You just experienced the full CI feedback loop. This is exactly what your teammates will see when they push broken code."

---

## Part 5 — Branch Protection (10 min)

### Turning CI from optional to required

> "Right now CI runs, but nothing stops you from merging a PR with a failed CI. Let's fix that."

**Live walkthrough in GitHub UI:**

1. Go to your repo → **Settings** → **Branches**
2. Click **Add branch protection rule**
3. Branch name pattern: `main`
4. Check: ✅ **Require status checks to pass before merging**
5. In the search box, type your job names: `lint`, `test`
6. Check: ✅ **Require branches to be up to date before merging**
7. Click **Save changes**

### Demonstrate it working

```bash
# Create a branch with a broken test
git checkout -b demo/broken-test
```

```javascript
// Break a test
test('broken', () => {
  expect(1).toBe(2);   // will fail
});
```

```bash
git add .
git commit -m "Broken test"
git push -u origin demo/broken-test
```

Go to GitHub → create a Pull Request from this branch to main.

> "Look at the PR. The merge button is greyed out with a message: 'Some checks were not successful.' The team is protected."

Fix the test, push again, watch the PR unblock.

```bash
# Undo the bad test
git add .
git commit -m "Fix test"
git push
```

---

## Part 6 — Stretch Goal: Deploy Step (10 min, if time permits)

> "For those who are ahead: let's add a deploy step that SSHes into your VM."

### Add a deploy job

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to VM
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/myapp
            git pull origin main
            npm ci --omit=dev
            pm2 restart myapp || pm2 start server.js --name myapp
```

### Add secrets to the repo

GitHub → Settings → Secrets and variables → Actions → New repository secret:

- `VM_HOST` — your VM's IP address or hostname
- `VM_USER` — SSH username (e.g., `ubuntu`, `ec2-user`)
- `SSH_PRIVATE_KEY` — contents of your `~/.ssh/id_rsa` private key

> "Your private key goes in GitHub Secrets. It's encrypted at rest and never shown in logs. GitHub replaces any accidental secret output with `***`."

---

## Wrap-Up and Debrief (5 min)

### What you built today

- ✅ ESLint configured and running locally
- ✅ GitHub Actions workflow with lint + test jobs
- ✅ Workflow runs automatically on push and PR
- ✅ Branch protection: PRs require green CI to merge
- ✅ (Stretch) Automated deploy to VM

### Questions to prompt discussion

1. "What would you add to this pipeline for your project?"
2. "Where in your current workflow would this have caught a bug?"
3. "What's the cost of a slow CI pipeline? How would you speed this one up?"

### Before Lab 2

- Install Docker Desktop: [docs.docker.com/get-docker](https://docs.docker.com/get-docker/)
- Run `docker run hello-world` to confirm it works
- Think about what services your app needs (database? cache?)

---

## Troubleshooting Guide

### "My workflow never runs"
- Check the `on:` trigger — does the branch name match exactly? (`main` vs `master`)
- Check the file is in `.github/workflows/` (two levels deep)
- Check the YAML is valid — use [yaml.org/start.html](https://yaml.org) or VS Code's YAML extension

### "npm ci fails but npm install works"
- `npm ci` requires a committed `package-lock.json`. Run `npm install` locally, commit the lock file.

### "Lint passes locally but fails in CI"
- CI uses a clean environment. Check that `.eslintrc` is committed.
- Check `node_modules/.bin/eslint` is not being relied on in a way that differs from `npm run lint`.

### "Test job runs even though lint failed"
- Make sure `needs: lint` is indented correctly under the `test` job, not under `steps`.

### "Secrets not available in the deploy step"
- Secrets are scoped to the repo where they're added. Confirm under Settings → Secrets.
- Forks do not get access to the parent repo's secrets (security feature).

### "Branch protection isn't blocking the merge"
- You need at least one workflow run to have completed before GitHub recognizes the check names. Push once, then set up protection.
