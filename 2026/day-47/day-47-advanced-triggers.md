# GitHub Actions Advanced Triggers: PR Events, Cron Schedules & Event-Driven Pipelines
## Task 1: Pull Request Event Types
### Create Workflow File `.github/workflows/pr-lifecycle.yml`:
```yaml
name: PR Lifecycle Events

on:
  pull_request:                                       #it runs when something happens to a Pull Request
    types: [opened, synchronize, reopened, closed]    #filters which PR events trigger it

jobs:
  pr-info:
    runs-on: ubuntu-latest
    steps:
      - name: Print event type                        #prints PR event type (e.g., opened, synchronize, etc.)
        run: echo "Event type: ${{ github.event.action }}"

      - name: Print PR title                          #prints title of PR
        run: echo "Title: ${{ github.event.pull_request.title }}"

      - name: Print PR author                         #prints GitHub username of person who opened PR
        run: echo "Author: ${{ github.event.pull_request.user.login }}"

      - name: Print branches
        run: echo "Source: ${{ github.head_ref }} -> Target: ${{ github.base_ref }}"

      - name: Run only if merged                      #this step runs only if PR was merged (not just closed)
        if: github.event.pull_request.merged == true
        run: echo "This PR was merged!"
```
- Conditional `if: github.event.pull_request.merged == true` ensures the merged message only prints for successful merges.
### Test the Workflow
Push Workflow to Default Branch
- Commit and push the workflow file to our repo’s default branch (`main` or `master`).
  ```bash
  git add .github/workflows/pr-lifecycle.yml
  git commit -m "Add PR Lifecycle Events workflow"
  git push origin main
  ```
- Verify on GitHub
  - Go to our repo’s **Actions tab**.
  - We should now see the workflow listed as **PR Lifecycle Events**.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(470).png)

  - Test it by opening a PR, pushing commits, reopening, and merging — each event will trigger the workflow.

Create a Pull Request
- Create a new branch:
  ```bash
  git checkout -b feature/test-pr-lifecycle
  ```
- Make a small change (e.g., edit `README.md`).
  ```bash
  # Edit README.md at the root of the repo
  echo "Testing PR lifecycle workflow" >> README.md

  # Commit and push
  git add README.md
  git commit -m "Update README to test PR lifecycle workflow"
  git push origin feature/test-pr-lifecycle
  ```
- Go to our repo on GitHub.
- We’ll see a banner suggesting open a PR for `feature/test-pr-lifecycle`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(472).png)

  - Click **Compare & pull request**.
- Select:
  - **Base branch:** `main`
  - **Compare branch:** `feature/test-pr-lifecycle`

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(474).png)

- Click **Create pull request**.
- As soon as we open the PR, the workflow fires with:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(476).png)

Synchronize Event
- Push another commit from our local repo on the same branch (`feature/test-pr-lifecycle`):
  ```bash
  # Make another small change
  echo "Second test line" >> README.md

  # Stage and commit
  git add README.md
  git commit -m "Second commit to trigger synchronize event"

  # Push to the same branch
  git push origin feature/test-pr-lifecycle
  ```
- Because the branch already has an open PR into `main`, pushing a new commit triggers the `synchronize` event.
- In the workflow logs (Actions tab ---> PR Lifecycle Events), we’ll see:
 
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(479).png)

Reopen Event
- Go to our Pull Request in GitHub.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(483).png)

- Click **Close pull request**.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(485).png)

- Then click **Reopen pull request**.
  - This action fires the `reopened` event in our workflow.
- In the **Actions tab --> PR Lifecycle Events --> Run details**, we’ll see logs like:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(487).png)

Closed + Merged Event
- Go to our PR in GitHub.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(489).png)

- Click **Merge pull request** ---> **Confirm merge**.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(490).png)

- The workflow runs again, this time with:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(492).png)

### What Happened?
- **opened** ---> when a new PR is created.
- **synchronize** ---> when commits are pushed to the PR branch.
- **reopened** ---> when a closed PR is reopened.
- **closed** ---> when a PR is closed (merged or not).

## Task 2: PR Validation Workflow
### Create the Workflow File `.github/workflows/pr-validation.yml`
```yaml
name: PR Validation Checks

on:
  pull_request:                               #runs whenever a Pull Request targets 'main' branch
    branches: [main]

jobs:
  file-size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4             #pulls our repo code into runner
      - name: Check file sizes
        run: |
          for f in $(git ls-files); do        #goes through all tracked files (git ls-files)
            size=$(stat -c%s "$f")            #(stat -c%s "$f") --> gets file size in bytes
            if [ $size -gt 1048576 ]; then    #if any file is larger than 1048576 bytes (1 MB), it prints a message and fails the job
              echo "File $f exceeds 1MB"
              exit 1
            fi
          done

  branch-name-check:
    runs-on: ubuntu-latest
    steps:
      - name: Validate branch name
        run: |
          BRANCH=${{ github.head_ref }}                       #(github.head_ref) --> gives source branch name of PR
          if [[ ! $BRANCH =~ ^(feature|fix|docs)/ ]]; then    #Regex check --> branch must start with feature/, fix/, or docs/
            echo "Branch name invalid: $BRANCH"
            exit 1
          fi
          echo "Branch name valid: $BRANCH"

  pr-body-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check PR body
        run: |
          BODY="${{ github.event.pull_request.body }}"        #(github.event.pull_request.body) --> contains PR description text


          if [ -z "$BODY" ]; then
            echo "Warning: PR description is empty"           #if empty --> prints a warning (but does not fail)
          else
            echo "PR description present"
          fi
```

### Test the Workflow
Push workflow files to `main` so GitHub Actions recognizes them.
```bash
git add .github/workflows/pr-validation.yml
git commit -m "Add PR Validation Checks workflow"
git push origin main
```
File Size Check
- Create a branch with a valid name (e.g., `feature/test-validation`).
- Add a file larger than 1MB at the root of the repo:
  ```bash
  dd if=/dev/zero of=bigfile.txt bs=2M count=1    #Linux command that creates a file filled with zeros
  git add bigfile.txt
  git commit -m "Add large file to test validation"
  git push origin feature/test-validation
  ```

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(500).png)

- Open a PR ---> workflow should fail with:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(502).png)

Branch Name Check
- Create a branch with a bad name:
  ```bash
  git checkout -b testbranch
  git push origin testbranch
  ```
- Open a PR into `main`.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(506).png)

- The workflow runs and the **branch-name-check** job fails with:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(508).png)

- Try again with a valid name (`feature/test-validation`):
  ```bash
  git checkout feature/test-validation
  ```
- Make a small change:
  ```bash
  echo "Testing valid branch" >> README.md
  git add README.md
  git commit -m "Test valid branch name"
  git push origin feature/test-validation
  ```
- Open a PR into `main`.
  - The workflow should pass with:
    
    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(511).png)

PR Body Check
- Open a PR without a description ---> workflow warns:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(512).png)

- Open a PR with a description(e.g., “This PR adds validation checks”). ---> workflow passes:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(514).png)

## Task 3: Scheduled Workflows (Cron Deep Dive)
### Create the Workflow File
Create a new file in our repo: `.github/workflows/scheduled-tasks.yml`
```yaml
name: Scheduled Tasks

on:
  schedule:
    - cron: '30 2 * * 1'   # Mondays 2:30 AM UTC
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:

jobs:
  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Print schedule
        run: echo "Triggered by schedule: ${{ github.event.schedule }}"

      - name: Health check
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://example.com)
          echo "Response code: $STATUS"
          if [ "$STATUS" -ne 200 ]; then
            echo "Health check failed"
            exit 1
          else
            echo "Health check passed"
          fi
```
### Notes 
- The cron expression for: **Every weekday at 9 AM IST**
    - IST = UTC+5:30 → 9:00 AM IST = 3:30 AM UTC
    - Cron:
        ```Code
        30 3 * * 1-5
        ```
- The cron expression for: **First day of every month at midnight(UTC)**
    ```Code
    0 0 1 * *
    ```
- **Why GitHub may delay/skip scheduled workflows on inactive repos?**<br>
**Ans.** GitHub Actions schedules are **best effort**. If a repository is inactive (no recent commits or workflow runs), scheduled jobs may be delayed or skipped to conserve resources. They only reliably run on the **default branch** of active repositories.

### Test the Workflow
- Use **workflow_dispatch** to trigger manually from the Actions tab.
- Verify the printed schedule and health check output.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(516).png)

- Wait for the cron schedules to fire automatically.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(520).png)

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(519).png)

## Task 4: Path & Branch Filters
### Create the Workflow File `.github/workflows/smart-triggers.yml`
```yaml
name: Smart Triggers

on:
  push:
    branches:                 #only triggers if push is to 'main' or any branch starting with 'release/'
      - main
      - release/*
    paths:                    #workflow only runs if files inside 'src/' or 'app/' change
      - 'src/**'
      - 'app/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Triggered by changes in src/ or app/"
```
Add a Second Workflow with paths-ignore: `.github/workflows/docs-ignore.yml`
```yaml
name: Docs Ignore

on:
  push:
    branches:
      - main
      - release/*
    paths-ignore:              #tells GitHub Actions to ignore pushes that only change - markdown files (*.md) & anything inside the docs/ folder
      - '*.md'
      - 'docs/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Triggered by code changes, not docs"
```
### Test the Workflows
- Create or edit a file inside `src/` or `app/` at the root of our repo:
  ```bash
  echo "console.log('test');" >> src/test.js
  git add src/test.js
  git commit -m "Trigger Smart Triggers workflow"
  git push origin main
  ```
- Go to the **Actions tab** --> we should see **Smart Triggers** & **Docs Ignore** run.
- The job from **Smart Triggers** prints:
  
  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(525).png)

- The job from **Docs Ignore** prints:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(524).png)

- Create or edit a `docs/` or `*.md` file:
  ```bash
  echo "Update docs" >> docs/guide.md
  git add docs/guide.md
  git commit -m "Update docs"
  git push origin main
  ```
  - The workflow will not run, because the changes are ignored.

- Now edit a code file:
  ```bash
  echo "print('hello')" >> app/test.py
  git add app/test.py
  git commit -m "Trigger Docs Ignore workflow"
  git push origin main
  ```
- The workflows runs and prints:

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(526).png)

### When would we use `paths` vs `paths-ignore`?
- **When to use `paths`:**  
Use when we want workflows to run **only for specific files or directories**. Example: run tests only when `src/` code changes.
- **When to use `paths-ignore`:**  
Use when we want workflows to run normally, but **skip for trivial changes** (like docs or config files). Example: skip CI when only `.`md` files are updated.

## Task 5: `workflow_run` — Chain Workflows Together
### Create the Test Workflow `.github/workflows/run-tests.yml`
```yaml
name: Run Tests

on:
  push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tests
        run: |
          echo "Running tests..."
          # Replace with our actual test command, e.g.:
          # pytest || exit 1
```
- This workflow runs **every time we push a commit**.
- It simulates a test job (replace with our real test suite).

### Create the Deploy Workflow `.github/workflows/deploy-after-tests.yml`
```yaml
name: Deploy After Tests

on:
  workflow_run:                                                                      #special trigger that listens for another workflow
    workflows: ["Run Tests"]                                                         #this workflow runs only after workflow named Run Tests completes
    types: [completed]                                                               #triggers regardless of whether Run Tests succeeded or failed

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Check triggering workflow
        run: |
          echo "Triggered by workflow: ${{ github.event.workflow_run.name }}"         #prints name of workflow that triggered this one
          echo "Conclusion: ${{ github.event.workflow_run.conclusion }}"              #prints conclusion (success, failure, cancelled)

          if [ "${{ github.event.workflow_run.conclusion }}" != "success" ]; then
            echo "Tests failed, skipping deploy"
            exit 1
          fi
          echo "Tests passed, proceeding with deploy..."
          # Add your deploy steps here, e.g.:
          # ./deploy.sh
```
- **Trigger:** Runs only after the `Run Tests` workflow completes.
- **Conditional:** Checks the conclusion of the test workflow.
    - If `success`, proceeds with deploy.
    - If not, prints a warning and exits.

### Verify the Chain
- Push a commit.
  ```bash
  git add .github/workflows/run-tests.yml .github/workflows/deploy-after-tests.yml
  git commit -m "Add Run Tests and Deploy After Tests workflows"
  git push origin main
  ```
  - This triggers **Run Tests**.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(528).png)

  - When Run Tests completes, GitHub automatically triggers **Deploy After Tests**.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(529).png)

- Simulate failure
  - Change the test step to something like:
    ```yaml
    - name: Run tests
      run: exit 1
    ```
  - Push a commit.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(531).png)

  - **Run Tests** fails --> **Deploy After Tests** runs, sees the failure, and exits with:
    
    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(530).png)

## Task 6: `repository_dispatch` — External Event Triggers
### Create the Workflow File `.github/workflows/external-trigger.yml`
```yaml
name: External Trigger

on:
  repository_dispatch:                #special event type that lets us trigger workflows manually via GitHub API or CLI
    types: [deploy-request]           #this workflow only runs when event type is deploy-request

jobs:
  external:
    runs-on: ubuntu-latest
    steps:
      - name: Print client payload
        run: echo "Environment: ${{ github.event.client_payload.environment }}"
```
- **Trigger:** `repository_dispatch` is a special event that external systems can fire.
- **Event type:** `deploy-request` ensures this workflow only runs when that specific event type is sent.
- **Payload:** We can pass custom JSON data (`client_payload`) and access it inside the workflow.

### Trigger the Workflow
We can trigger this workflow using the GitHub CLI (`gh`) or `curl`.<br>

```bash
gh api repos/atulsharmadevops/github-actions-practice/dispatches \
  -f event_type=deploy-request \
  -F client_payload[environment]=production
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(532).png)

Or with `curl`:
```bash
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer personal-access-token" \
  https://api.github.com/repos/atulsharmadevops/github-actions-practice/dispatches \
  -d '{"event_type":"deploy-request","client_payload":{"environment":"staging"}}'
```
- Requires a **Personal Access Token** with `repo` scope.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/83aa9e80b1d728ed8b60eb9e0238fa6d04550070/2026/day-47/Screenshots/Screenshot%20(534).png)

### When would an external system (like a Slack bot or monitoring tool) trigger a pipeline?<br>
**Ans.**

- A **Slack bot** could trigger a deploy when a team member types `/deploy`.
- A **monitoring tool** could trigger a rollback workflow if it detects downtime.
- A **release management system** could trigger builds when a new version is tagged outside GitHub.
- A **CI/CD orchestrator** could trigger GitHub Actions as part of a larger pipeline.
