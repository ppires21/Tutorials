# From Zero to DevOps with GitHub Actions
### A Practical Step-by-Step Guide using a Real Python Project (OpenWeather API Example)

---

## Introduction — What is GitHub Actions?

**GitHub Actions** is a tool built into GitHub that allows you to **automate tasks** such as testing, building, and deploying your code.  
Every automation is defined in a file called a **workflow**, written in **YAML** format.

A workflow can be triggered automatically — for example:
- When you push code to a branch
- When a pull request is opened
- Or even on a schedule (like every Monday at 8 AM)

Each workflow runs one or more **jobs**. Each job runs a sequence of **steps** on a **runner** (a virtual machine provided by GitHub).

Let’s visualize the flow:

```
Trigger (push, PR, schedule)
        ↓
   Workflow (YAML file)
        ↓
     Jobs (parallel)
        ↓
    Steps (sequential commands)
        ↓
     Runner executes them
```

You’ll now learn GitHub Actions progressively through **4 real exercises**.

---

# Exercise 1 — Your First Workflow

### Goal
Create your very first GitHub Action that simply prints text and lists files when you push to a branch.

This is the “Hello World” of DevOps.

---

### File Location
Create a new file in your repository:
```
.github/workflows/exercise1.yml
```

### Step-by-step Explanation

#### 1️⃣ Workflow name
```yaml
name: Exercise 1 - Hello World
```
- This is just a **label** for the workflow. It appears in GitHub under the “Actions” tab.

#### 2️⃣ Trigger (when it runs)
```yaml
on:
  push:
    branches: ["ex1"]
```
- `on:` defines what triggers this workflow.  
- Here, it runs when you **push to the branch `ex1`**.

#### 3️⃣ The job section
```yaml
jobs:
  hello-world:
    runs-on: ubuntu-latest
```
- `jobs:` starts the list of jobs.
- `hello-world:` is the job name.
- `runs-on:` defines the **runner environment** — here, a fresh **Ubuntu Linux VM** provided by GitHub.

#### 4️⃣ The steps
```yaml
    steps:
      - name: Checkout the code
        uses: actions/checkout@v4
```
- `steps:` defines the sequence of actions to run.  
- The first step uses a **pre-built GitHub Action** (`actions/checkout`) to clone your code into the runner.

```yaml
      - name: Print Hello
        run: echo "Hello, GitHub Actions!"
```
- `run:` executes a shell command.

```yaml
      - name: List files
        run: ls -la
```
- Lists the files that were cloned.

---

**Test it!**
1. Create a branch named `ex1`.
2. Commit and push this file.
3. Go to the “Actions” tab in your repository → you’ll see it run.

Congratulations — you’ve just built your first workflow! 🎉

---

# Exercise 2 — Continuous Integration for a Python Project

### Goal
Set up a Python environment, install dependencies, run linting (flake8), execute tests (pytest), and use a **secret** for API access.

---

### 1️⃣ The concept of a “runner”
Each workflow runs inside a temporary virtual machine (VM) called a **runner**.  
When your workflow starts, you get a clean Linux environment every time.

We’ll now tell the runner how to:
- Install Python
- Set up dependencies
- Run tests and scripts

---

### 2️⃣ Create a secret in GitHub
1. Go to your repo → **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name it: `WEATHER_API_KEY`
4. Paste your real OpenWeather API key.

This secret will be available only inside workflows as `secrets.WEATHER_API_KEY`.

---

### 3️⃣ Create the workflow file

File: `.github/workflows/exercise2.yml`

```yaml
name: Exercise 2 - Python CI

on:
  push:
    branches: ["ex2"]
```

#### Breakdown
- Triggers on any push to the branch `ex2`.
- The name will appear in the Actions UI.

---

### 4️⃣ Define the job

```yaml
jobs:
  python-ci:
    runs-on: ubuntu-latest
    env:
      ENV_PATH: "./venvs"
      WEATHER_API_KEY: ${{ secrets.WEATHER_API_KEY }}
```

- Creates one job called `python-ci`.
- Sets an environment variable `ENV_PATH` for the virtual environment location.
- Injects your secret `WEATHER_API_KEY` into the job’s environment.

---

### 5️⃣ Add the steps

#### a) Checkout
```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```
→ This pulls your repository into the runner so it can be accessed.

#### b) Install Python
```yaml
      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
```
→ Downloads and configures the specified Python version.

#### c) Create and activate virtual environment
```yaml
      - name: Create virtual environment
        run: python -m venv $ENV_PATH/.venv
```

#### d) Install dependencies
```yaml
      - name: Install dependencies
        run: |
          . $ENV_PATH/.venv/bin/activate
          pip install -U pip
          pip install -r openweatherapiclient/requirements.txt
        shell: bash
```

#### e) Lint the code
```yaml
      - name: Run flake8
        run: |
          . $ENV_PATH/.venv/bin/activate
          flake8 openweatherapiclient
        shell: bash
```

#### f) Run tests
```yaml
      - name: Run tests
        run: |
          . $ENV_PATH/.venv/bin/activate
          pytest -q openweatherapiclient/tests/main_tests.py
        shell: bash
```

#### g) Execute the main script
```yaml
      - name: Run the script with secret
        env:
          CITY_TO_CALL: "Porto"
        run: |
          . $ENV_PATH/.venv/bin/activate
          python openweatherapiclient/weatherapi/main.py $CITY_TO_CALL
        shell: bash
```

---

**What you’ve learned**
- How to use GitHub’s hosted runners
- How to install dependencies inside Actions
- How to access **repository secrets**
- How to run linting and tests automatically

You now have **Continuous Integration (CI)** — every push ensures your project builds and tests successfully!

---

# Exercise 3 — Reusable Actions and Matrix Builds

### Goal
Avoid repeating setup steps and test your project on multiple Python versions.

---

### 1️⃣ Why reuse?
In Exercise 2, you wrote a lot of repetitive setup code.  
We’ll now create a **custom reusable action** that prepares your Python environment.

---

### 2️⃣ Create a composite action

File: `.github/actions/setup-python-env/action.yml`

```yaml
name: "Setup Python Environment"
description: "Reusable setup with venv + dependencies + cache"

inputs:
  python-version:
    required: true
  env-path:
    required: true

runs:
  using: "composite"
  steps:
    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - name: Create virtual environment
      run: python -m venv ${{ inputs.env-path }}/.venv
      shell: bash

    - name: Install dependencies
      run: |
        source ${{ inputs.env-path }}/.venv/bin/activate
        pip install -U pip
        pip install -r openweatherapiclient/requirements.txt
      shell: bash
```

This is your **own custom Action**.  
It can be reused in any workflow within your repo.

---

### 3️⃣ Use the custom action in a workflow

File: `.github/workflows/exercise3.yml`

```yaml
name: Exercise 3 - Matrix and Composite

on:
  push:
    branches: ["ex3"]
```

#### a) Define the matrix job
```yaml
jobs:
  test-matrix:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11"]
```

- The **matrix** runs this job once per Python version.

#### b) Steps
```yaml
    steps:
      - uses: actions/checkout@v4

      - name: Setup environment
        uses: ./.github/actions/setup-python-env
        with:
          python-version: ${{ matrix.python-version }}
          env-path: "./venvs"

      - name: Run tests
        run: |
          . ./venvs/.venv/bin/activate
          pytest -q openweatherapiclient/tests/main_tests.py
        shell: bash
```

Each Python version runs the same tests. A mini compatibility check!

---

# Exercise 4 — Automating Dependency Updates and Pull Requests

### Goal
Automatically check for outdated dependencies weekly, update them, and open a Pull Request with the changes.

---

### Step-by-step Explanation

#### 1️⃣ Schedule or run manually
```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: "0 6 * * 1"  # every Monday at 6 AM UTC
```
- `workflow_dispatch`: allows manual trigger.
- `schedule`: sets up an automatic cron job.

#### 2️⃣ Permissions
```yaml
permissions:
  contents: write
  pull-requests: write
```
→ required to commit and create PRs.

#### 3️⃣ Update logic
The job will:
1. Create a virtual environment
2. Install pip-tools
3. Check for outdated dependencies
4. If outdated → regenerate `requirements.txt`
5. Commit and open a PR automatically

Simplified version:

```yaml
jobs:
  deps-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Setup environment
        run: |
          python -m venv .venv
          source .venv/bin/activate
          pip install -U pip pip-tools
          pip install -r openweatherapiclient/requirements.txt

      - name: Update requirements
        run: |
          source .venv/bin/activate
          pip-compile openweatherapiclient/requirements.in -o openweatherapiclient/requirements.txt

      - name: Commit and push changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git checkout -b deps-update
          git add openweatherapiclient/requirements.txt
          git commit -m "chore: update dependencies"
          git push origin deps-update

      - name: Open Pull Request
        run: gh pr create --title "Update dependencies" --body "Auto update" --base main --head deps-update
        env:
          GH_TOKEN: ${{ github.token }}
```

---

**Result**
- Every week, this job automatically updates your dependencies.
- If updates exist, a new PR is opened with all changes.
- You just review and merge!

---

# Final Recap

| Exercise | Concept | What You Learned |
|-----------|----------|------------------|
| 1 | Basic workflow | triggers, jobs, steps |
| 2 | Python CI | virtual env, dependencies, secrets |
| 3 | Reusable logic | composite actions, matrix builds |
| 4 | Automation | scheduled tasks, pull requests, pip-tools |

---

🎉 **You now know how to build, test, and automate a Python project using GitHub Actions!**
You can extend these workflows to deploy code, build Docker images, or publish packages.

Next step? Try combining everything into a single workflow for a full CI/CD pipeline.

