# CI/CD with GitHub Actions — In-class Exercise

This exercise walks you through building a CI/CD pipeline step by step. You start with the source code only — **no workflow files are provided**. You will create each workflow manually, push it, and watch its status badge change from failing to green in the repository's README.

## Prerequisites
- GitHub account
- Basic Git knowledge
- Git installed locally

---

## Step 1 — Get your own copy of the repository

### Option A — Double remote (recommended)

1. **Create an empty repository** in your GitHub account:
   - Go to [github.com/new](https://github.com/new)
   - Give it any name (e.g., `CICDTaskManager`)
   - Leave *"Add a README"* and all other options **unchecked**
   - Click *"Create repository"*

2. **Clone the course repository** to your machine:
   ```bash
   git clone https://github.com/IyPSUMA/CICDTaskManager.git
   cd CICDTaskManager
   ```

3. **Add your own repository as a second remote:**
   ```bash
   git remote add mygithub https://github.com/<your-username>/CICDTaskManager.git
   ```
   You now have two remotes:
   - `origin` — the course repository (read-only for you)
   - `mygithub` — your own copy on GitHub

### Option B — Fork

Alternatively, you can **fork** the repository directly from GitHub:

1. Go to [github.com/IyPSUMA/CICDTaskManager](https://github.com/IyPSUMA/CICDTaskManager)
2. Click the **Fork** button (top right)
3. Clone your fork locally:
   ```bash
   git clone https://github.com/<your-username>/CICDTaskManager.git
   cd CICDTaskManager
   ```

> A fork creates a linked copy under your account. This will be useful later when we cover **pull requests**.

### Configure badges and push

4. **Update the badge URLs in `README.md`** — replace the placeholders with your own username and repo name:
   ```
   YOUR_GITHUB_USERNAME  →  your GitHub username
   YOUR_REPO_NAME        →  CICDTaskManager  (or whatever you named it)
   ```

5. **Push the code to your repository:**
   ```bash
   git add README.md
   git commit -m "Set badge URLs"
   git push -u mygithub main       # if using double remote
   # git push origin main           # if using fork
   ```

Go to your repository on GitHub and open the **README** tab. You will see the badges — they will show as failing or unknown because no workflow files exist yet. That is expected. They will turn green as you add each workflow in the steps below.

---

## Step 2 — Create the workflows directory

All GitHub Actions workflow files live in `.github/workflows/`. Create this directory:

```bash
mkdir -p .github/workflows
```

---

## Step 3 — Workflow 1: Compile

**What it does:** Checks out the code and compiles it with Maven. This is the fastest feedback: if the code doesn't compile, this workflow fails immediately.

Create the file `.github/workflows/compile.yml` with this content:

```yaml
name: Compile

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Compile
        run: mvn compile --batch-mode
```

Push and observe:
```bash
git add .github/workflows/compile.yml
git commit -m "Add compile workflow"
git push mygithub main
```

Go to the **Actions** tab and watch the `Compile` job run. Once it passes, the **Compile** badge in the README turns green.

---

## Step 4 — Workflow 2: Test

**What it does:** Runs all unit tests (files matching `*Test.java`) with Maven Surefire. Uploads the Surefire XML reports as an artifact so they are available from the run summary page.

Create `.github/workflows/test.yml`:

```yaml
name: Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Run unit tests
        run: mvn test --batch-mode
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
```

> Note `if: always()` on the upload step — test results are uploaded even when some tests fail, which is exactly when you most want them.

```bash
git add .github/workflows/test.yml
git commit -m "Add test workflow"
git push mygithub main
```

---

## Step 5 — Workflow 3: Build

**What it does:** Compiles, runs tests, and packages the project into a JAR. Also generates Javadoc and uploads both as artifacts.

Create `.github/workflows/build.yml`:

```yaml
name: Build

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Build with Maven
        run: mvn -B clean package --file pom.xml

      - name: Generate JavaDoc
        run: mvn javadoc:javadoc

      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

      - name: Upload JavaDoc artifact
        uses: actions/upload-artifact@v4
        with:
          name: javadoc
          path: target/site/apidocs
```

```bash
git add .github/workflows/build.yml
git commit -m "Add build workflow"
git push mygithub main
```

After it passes, download the JAR from the run summary and inspect it.

---

## Step 6 — Workflow 4: Integration Test

**What it does:** Runs Maven's `integration-test` lifecycle phase, which picks up test files matching `*IT.java` (Mockito-based top-down tests in `src/test/.../integration/`).

Create `.github/workflows/integration-test.yml`:

```yaml
name: Integration Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  integration-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Run integration tests
        run: mvn integration-test --batch-mode
```

```bash
git add .github/workflows/integration-test.yml
git commit -m "Add integration-test workflow"
git push mygithub main
```

---

## Step 7 — Workflow 5: Javadoc

**What it does:** Generates the Javadoc API documentation independently from the build and uploads it as an artifact.

Create `.github/workflows/javadoc.yml`:

```yaml
name: Javadoc

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  javadoc:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Generate Javadoc
        run: mvn javadoc:javadoc --batch-mode
      - name: Upload Javadoc
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: javadoc
          path: target/site/apidocs/
```

```bash
git add .github/workflows/javadoc.yml
git commit -m "Add javadoc workflow"
git push mygithub main
```

All five badges in the README should now be green.

---

## Step 8 — Single Job vs Multi Job

Now that you understand the individual workflows, compare two alternative approaches.

### Option A — Single Job

All stages run sequentially as steps within one job. Simpler structure, single badge.

Create `.github/workflows/ci-single-job.yml`:

```yaml
name: CI (Single Job)

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven

      - name: Compile
        run: mvn compile --batch-mode

      - name: Run unit tests
        run: mvn test --batch-mode

      - name: Build JAR
        run: mvn package --batch-mode

      - name: Run integration tests
        run: mvn integration-test --batch-mode

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/

      - name: Upload JAR artifact
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
```

### Option B — Multi Job

Each stage is an independent job with explicit `needs:` dependencies. Compile must pass before Test runs; Test must pass before Build; Build before Integration Test. Coverage and Javadoc run in parallel after their respective dependencies.

Create `.github/workflows/ci-multi-job.yml`:

```yaml
name: CI (Multi Job)

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Compile
        run: mvn compile --batch-mode

  test:
    runs-on: ubuntu-latest
    needs: compile
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Run unit tests
        run: mvn test --batch-mode
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Build JAR
        run: mvn -B clean package
      - name: Upload JAR artifact
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  integration-test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Run integration tests
        run: mvn integration-test --batch-mode

  coverage:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Run JaCoCo coverage
        run: mvn test -P coverage --batch-mode
      - name: Upload JaCoCo report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: target/site/jacoco/

  javadoc:
    runs-on: ubuntu-latest
    needs: compile
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Generate Javadoc
        run: mvn javadoc:javadoc --batch-mode
      - name: Upload Javadoc
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: javadoc
          path: target/site/apidocs/
```

```bash
git add .github/workflows/ci-single-job.yml .github/workflows/ci-multi-job.yml
git commit -m "Add single-job and multi-job CI workflows"
git push mygithub main
```

Go to the **Actions** tab and compare:
- In `CI (Single Job)`, all steps are listed inside a single job node.
- In `CI (Multi Job)`, the job graph shows the dependency chain visually. If `compile` fails, all downstream jobs are skipped automatically.

---

## Step 9 — Break the pipeline (optional)

Introduce a compile error and push to observe which jobs fail and which are skipped:

```bash
# Add a syntax error in any .java file, then:
git add .
git commit -m "Introduce compile error"
git push mygithub main
```

Watch the `ci-multi-job.yml` run: `compile` will fail and all jobs that `needs: compile` will be skipped. Revert the change and push again to restore the green state.
