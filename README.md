# CHARUSAT ITUE301 — Practical 12: CI/CD Pipeline with GitHub Actions

[![CI Pipeline](https://github.com/shahparam15/24CS093_AWDF_Practical-12/actions/workflows/ci.yml/badge.svg)](https://github.com/shahparam15/24CS093_AWDF_Practical-12/actions/workflows/ci.yml)

**Course:** Advanced Web Development Frameworks (ITUE301)  
**Student Name:** Param Shah  
**Student ID:** 24CS093  
**Email:** 24cs093@charusat.edu.in  
**Department:** Computer Science & Engineering, CSPIT, CHARUSAT  
**Repository:** [https://github.com/shahparam15/24CS093_AWDF_Practical-12](https://github.com/shahparam15/24CS093_AWDF_Practical-12)

---

## 1. Overview & Objectives

Practical 12 implements an enterprise-grade Continuous Integration (CI) pipeline using **GitHub Actions** for the full-stack Task Management application. The CI workflow ensures high code quality, automated test coverage, and repeatable production build verification across all code changes pushed to the repository.

### Key Objectives
1. **Automated Code Linting:** Validate JavaScript/React syntax and formatting across the frontend repository (`npm run lint`).
2. **Automated Backend Integration Testing:** Execute the backend test suite (`node --test`) against an ephemeral **MongoDB 7.0 service container** inside the GitHub Actions runner.
3. **Frontend Production Build Verification:** Ensure Vite bundle compilation succeed without unresolved assets or TypeScript/Babel errors (`npm run build`).
4. **Trigger Matrix:** Automated pipeline execution on all `push` and `pull_request` events targeting the `main` branch, plus manual runs via `workflow_dispatch`.

---

## 2. GitHub Actions Workflow Configuration

The workflow is defined in [`.github/workflows/ci.yml`](.github/workflows/ci.yml):

```yaml
name: Task Manager CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  lint:
    name: Lint & Code Style
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install Frontend Dependencies
        working-directory: frontend
        run: npm ci

      - name: Run Linter
        working-directory: frontend
        run: npm run lint

  test-backend:
    name: Backend Integration Tests
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:7.0
        ports:
          - 27017:27017
        options: >-
          --health-cmd mongosh
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install Backend Dependencies
        working-directory: backend
        run: npm ci

      - name: Run Backend Integration Tests
        working-directory: backend
        env:
          PORT: 5000
          NODE_ENV: test
          MONGO_URI: mongodb://127.0.0.1:27017/task_manager_ci_test
          JWT_SECRET: super_secret_jwt_key_charusat_itue301_ci_testing
          CLIENT_ORIGIN: http://localhost:5173
        run: npm test

  build-frontend:
    name: Frontend Build Verification
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install Frontend Dependencies
        working-directory: frontend
        run: npm ci

      - name: Build Vite Production Bundle
        working-directory: frontend
        run: npm run build
```

---

## 3. Deliberate Failure and Recovery Demonstration

To demonstrate that the CI pipeline genuinely prevents broken code from entering `main`:

### Step 1: Inducing a Deliberate Failure
1. Edit `backend/test/api.test.js`:
   Change the expected HTTP status code of `GET /` or `GET /tasks` from `200` to `418` (I'm a teapot):
   ```js
   assert.strictEqual(res.status, 418); // Deliberate failure
   ```
2. Commit and push the broken branch:
   ```bash
   git checkout -b test-ci-failure
   git commit -am "test: deliberate assertion failure to verify CI"
   git push origin test-ci-failure
   ```
3. Open a Pull Request into `main`.
4. **Result:** The `test-backend` job fails immediately with `AssertionError [ERR_ASSERTION]: Expected values to be strictly equal: 200 !== 418`. GitHub marks the PR with a red `X` and blocks merging.

### Step 2: Fixing and Verifying Recovery
1. Revert the assertion back to `200`:
   ```js
   assert.strictEqual(res.status, 200); // Fixed
   ```
2. Commit and push the fix:
   ```bash
   git commit -am "fix: restore status code assertion to 200"
   git push origin test-ci-failure
   ```
3. **Result:** GitHub Actions triggers automatically, all three jobs (`lint`, `test-backend`, `build-frontend`) pass with green checkmarks, and the CI status badge shows `passing`.

---

## 4. Local Execution & Validation

You can validate each CI stage locally on your workstation before pushing:

### 1. Frontend Linting
```bash
npm --prefix frontend run lint
```

### 2. Backend Integration Tests
Ensure local MongoDB is running (`net start MongoDB` or local daemon):
```bash
npm --prefix backend test
```

### 3. Frontend Production Build
```bash
npm --prefix frontend run build
```

---

## 5. Repository File Structure

```text
24CS093_AWDF_Practical-12/
├── .github/
│   └── workflows/
│       └── ci.yml               # GitHub Actions CI Workflow Definition
├── backend/
│   ├── Dockerfile               # Node.js backend container definition
│   ├── package.json             # Backend dependencies & test scripts
│   ├── src/
│   │   ├── config/cache.js      # In-memory TTL cache (Practical 9)
│   │   ├── events/              # EventEmitter & background listeners (Practical 10)
│   │   ├── controllers/         # Auth & Task controllers
│   │   ├── routes/              # Express API route handlers
│   │   └── server.js            # Express server entry point
│   └── test/
│       └── api.test.js          # Automated integration test suite
├── frontend/
│   ├── Dockerfile               # Multi-stage Vite + Nginx build
│   ├── nginx.conf               # Production Nginx SPA reverse proxy
│   ├── package.json             # Frontend dependencies & build scripts
│   └── src/                     # React 19 application components & pages
├── docker-compose.yml           # Multi-service composition (Practical 11)
├── package.json                 # Root project scripts
└── README.md                    # Practical 12 documentation
```
