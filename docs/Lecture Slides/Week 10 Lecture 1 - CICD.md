---
share_cis4004: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---
# CI/CD with GitHub Actions
### Automating the path from code to deployment

*"If it's not automated, it's a liability."*

---

##   The Problem We're Solving

### Every team eventually hits this wall

You've been here (or you will be):

- "It works on my machine"  but breaks on the server
- Forgetting to run tests before pushing
- Manual deploys that only one person knows how to do
- A bug slips to production because the review was rushed
- Merge conflicts that break everything and nobody knows why

**CI/CD is the systematic answer to all of these.**

---

##   What Is CI/CD?

### Two ideas, often combined

| Term | Stands For | What It Means |
|------|-----------|----------------|
| **CI** | Continuous Integration | Automatically build and test every code change |
| **CD** | Continuous Delivery / Deployment | Automatically deliver tested code to an environment |

**The key word is *automatically*.**

CI/CD replaces "someone remembered to do it" with "the system always does it."

---

##   The CI/CD Pipeline "assembly line"

<style>
.reveal pre {
	font-size: 0.8em;
}
</style>
```
Developer pushes code
        │
        ▼
  ┌─────────────┐
  │  Trigger    │  ← something happened (push, PR, etc.)
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Build     │  ← install deps, compile, lint
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │    Test     │  ← unit tests, integration tests
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Deploy    │  ← push to server, cloud, container
  └─────────────┘
```

Each stage is a gate. **Failure stops the line.**

---

##   Why CI/CD Changes Team Dynamics

### It's not just automation  it's trust

**Without CI/CD:**
- "Did you test this?" is a social question
- One person holds deployment knowledge
- Broken main branch = everyone is blocked

**With CI/CD:**
- Tests run on every push, automatically
- Any team member can deploy safely
- Broken builds are caught before they affect others
- Code review is about logic, not "did you remember to..."

> CI/CD turns good intentions into enforced process.

---

##   GitHub Actions: The Basics

### GitHub's built-in CI/CD platform

GitHub Actions is:
- **Free** for public repos; generous free tier for private
- **Deeply integrated** with your repo (PRs, branches, releases)
- **YAML-based**  your pipeline lives in your repo as code
- **Extensible**  thousands of community "Actions" to reuse

Key vocabulary:

| Term | Meaning |
|------|---------|
| **Workflow** | A YAML file that defines your automation |
| **Trigger (on:)** | What event starts the workflow |
| **Job** | A group of steps that run on one machine |
| **Step** | A single command or Action |
| **Runner** | The virtual machine that executes the job |

---

##   Anatomy of a Workflow File

### Everything lives in `.github/workflows/`

```yaml
name: CI                        # shown in GitHub UI

on:                             # TRIGGER
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:                           # JOBS
  test:                         # job name (you pick it)
    runs-on: ubuntu-latest      # runner OS

    steps:                      # STEPS (run in order)
      - uses: actions/checkout@v4          # checkout your code
      - uses: actions/setup-node@v4        # install Node.js
        with:
          node-version: '20'
      - run: npm ci              # install deps
      - run: npm test            # run your tests
```

**This file is version-controlled just like your code.**

---

####  Triggers: When Does a Workflow Run?

```yaml
# On push to specific branches
on:
  push:
    branches: [main, develop]

# On pull requests targeting main
on:
  pull_request:
    branches: [main]

# On a schedule (cron syntax)
on:
  schedule:
    - cron: '0 9 * * 1'   # Every Monday at 9am UTC

# Manually triggered from GitHub UI
on:
  workflow_dispatch:

# Multiple triggers together
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
```

---

####  Jobs can run in parallel or in sequence

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint           # ← waits for lint to pass first
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test

  deploy:
    runs-on: ubuntu-latest
    needs: [lint, test]   # ← waits for BOTH
    if: github.ref == 'refs/heads/main'
    steps:
      - run: echo "Deploy would happen here"
```

---

## The `uses` vs `run` Distinction

**`run:`**  executes a shell command directly

```yaml
- run: npm ci
- run: npm test
- run: |
    echo "Multi-line"
    echo "shell script"
```

---

**`uses:`**  runs a pre-built Action from the marketplace

```yaml
- uses: actions/checkout@v4          # checks out your repo
- uses: actions/setup-node@v4        # configures Node.js
  with:
    node-version: '20'
- uses: actions/upload-artifact@v4   # saves files for later
  with:
    name: test-results
    path: ./coverage
```

> Actions are reusable building blocks. Think of them as npm packages for your pipeline.

---

##   Secrets and Environment Variables

### Never hardcode credentials

**Secrets**  stored encrypted in GitHub, injected at runtime:

```yaml
# In GitHub: Settings → Secrets and variables → Actions
# Add: DEPLOY_KEY, DATABASE_URL, API_TOKEN, etc.

steps:
  - run: ./deploy.sh
    env:
      DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
      DEPLOY_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

**Environment variables**  for non-sensitive config:

```yaml
env:
  NODE_ENV: test
  PORT: 3000

jobs:
  test:
    env:
      DATABASE_URL: postgres://localhost/testdb  # job-level
    steps:
      - run: npm test
        env:
          LOG_LEVEL: debug  # step-level
```

**Rule: if it's a secret, it goes in Secrets. Never in your YAML file.**

---

##   Pull Requests as Quality Gates

### PRs + CI = enforced standards

Without CI on PRs:
```
developer → pushes branch → reviewer eyeballs code → merges → 💥 breaks main
```

With CI on PRs:
```
developer → pushes branch → CI runs automatically
    → tests pass? reviewer sees green checkmark
    → tests fail? PR is blocked until fixed
    → reviewer focuses on logic, not mechanics
```

**You can require CI to pass before merging:**
- Repo Settings → Branches → Branch protection rules
- ✅ Require status checks to pass before merging
- Select your workflow job name

This turns CI from "nice to have" into "enforced team standard."

---

<style>
.reveal pre {
	font-size: 0.75em;
}
</style>
```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]   # test on multiple versions

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'               # cache node_modules

      - run: npm ci
      - run: npm run build --if-present
      - run: npm test
```

---

##   Deployment from GitHub Actions

For your projects (deploying to a cloud VM), a typical pattern:

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main'   # only deploy from main

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to VM via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            git pull origin main
            npm ci --production
            pm2 restart myapp
```

**Your server pulls the latest code. GitHub Actions is the trigger.**

---

##   Viewing Results in GitHub

### Where to find CI/CD feedback

1. **Actions tab** → all workflow runs, full logs per step
2. **Pull Request page** → inline pass/fail status with links
3. **Commit list** → green ✅ or red ❌ next to each commit
4. **Email notifications** → if a workflow you triggered fails

**Reading logs:**
- Each step is collapsible
- Failed steps show the exact command and output
- Exit codes matter: `0` = success, anything else = failure
- Timestamps help diagnose slow steps

---

##   Common Mistakes and How to Avoid Them

### Things that trip people up

| Mistake                        | Fix                                                        |
| ------------------------------ | ---------------------------------------------------------- |
| Workflow never runs            | Check the `on:` trigger matches your branch name exactly   |
| `npm test` exits with no tests | Add a test script to `package.json`                        |
| Secrets not available          | Confirm they're added under repo Settings, not org-level   |
| Deploy runs on every PR        | Add `if: github.ref == 'refs/heads/main'`                  |
| Works locally, fails in CI     | CI uses a clean env  check you committed all config files  |
| Slow workflows                 | Use `cache: 'npm'` in setup-node, split into parallel jobs |

---

##   CI/CD Best Practices

### Rules to build good habits

1. **Keep workflows fast**  slow CI gets ignored or bypassed; aim for under 5 minutes
2. **Fail fast**  run cheap checks (lint) before expensive ones (integration tests)
3. **Pin Action versions**  use `@v4` not `@main` to avoid surprise breakage
4. **One workflow per concern**  separate CI from CD for clarity
5. **Test your workflow changes in a branch**  don't debug on main
6. **Make the green checkmark meaningful**  a CI that always passes teaches nothing

---

##   What's Coming in the Lab

### Demo 1 (60 min): Hands-on GitHub Actions

You will:
1. Add a `ci.yml` workflow to your project repo
2. Write a test and watch it run automatically on push
3. Add a lint step using ESLint
4. Set up branch protection so PRs require green CI
5. Intentionally break a test  observe the PR block in action
6. *(Stretch)* Add a deploy step that SSHes into a VM

**Come with:** your project repo on GitHub, Node.js project with at least one test

---

##   Summary

### What we covered

- **CI** = automatically build and test every change
- **CD** = automatically deliver passing code to an environment
- **GitHub Actions** = YAML workflows that live in your repo
- **Triggers** control when workflows run (push, PR, schedule)
- **Jobs** group steps; jobs can depend on each other
- **Secrets** keep credentials out of your code
- **Branch protection** turns CI from optional to enforced

**Next up:** Lecture 2  Docker for Collaborative Development

*Before Lab 1:* make sure your repo has `npm test` working locally.

---

## Speaker Notes

###  (Pipeline Mental Model)
Draw this on the board first, then show the slide. Ask: "Where does this break down in your current workflow?" Get 2-3 answers before moving on.

###  (Jobs and Steps)
Emphasize `needs:`  this is the dependency graph. A deploy job that doesn't `needs: test` will deploy broken code. This is a real bug teams make.

###  (Pull Requests as Quality Gates)
This is the cultural shift, not just the technical one. CI enforces team norms. Ask: "Have you ever merged something that broke main? What would have stopped that?"

###  (Deployment)
Walk through the SSH pattern slowly. Students deploying to VMs will use exactly this. Point out that the server is *pulling*  the VM needs to have the repo cloned already.

###  (Common Mistakes)
Leave 5 minutes here. These are the bugs they'll hit in Lab 1. Inoculate them now.
