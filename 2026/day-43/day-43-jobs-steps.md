# GitHub Actions - Jobs, Steps, Env Vars & Conditionals

## Task 1: Multi-Job Workflow
### Create a File: `.github/workflows/multi-job.yml`
```yaml
name: Multi Job Workflow

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: echo "Building the app"

  test:
    runs-on: ubuntu-latest
    needs: build   # test waits for build
    steps:
      - name: Test
        run: echo "Running tests"

  deploy:
    runs-on: ubuntu-latest
    needs: test    # deploy waits for test
    steps:
      - name: Deploy
        run: echo "Deploying"
```
### Verification
- Push this file to our `github-actions-practice` repo.
- Go to the **Actions** tab ---> select the workflow run.
- In the workflow graph, we should see the dependency chain:
**Build ---> Test ---> Deploy**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(367).png)

This confirms that the jobs are running in sequence, with each depending on the success of the previous one.

## Task 2: Environment Variables
### File: `.github/workflows/env-vars.yml`
```yaml
name: Environment Variables Demo

on: [push]

# Workflow-level environment variable
env:  #declares environment variables available to all jobs and steps in this workflow
  APP_NAME: myapp #sets variable called APP_NAME with value myapp



jobs:
  demo:
    runs-on: ubuntu-latest
    # Job-level environment variable
    env:  #declares environment variables available to all steps in this job only
      ENVIRONMENT: staging  #sets variable called ENVIRONMENT with the value staging

yaml
    steps:
      - name: Print all variables
        # Step-level environment variable
        env:  #declares environment variables available only in this step.
          VERSION: 1.0.0  #sets variable called VERSION with the value 1.0.0.
        run: |
          # Print workflow-level variable
          echo "Workflow-level APP_NAME: $APP_NAME"

          # Print job-level variable
          echo "Job-level ENVIRONMENT: $ENVIRONMENT"

          # Print step-level variable
          echo "Step-level VERSION: $VERSION"

          # Print commit SHA from GitHub context
          echo "Commit SHA: ${{ github.sha }}"

          # Print actor (username who triggered workflow)
          echo "Actor: ${{ github.actor }}"
```
### Verification
- Push this workflow to our repo.
- Trigger a run (e.g., by committing a change).
- In the **Actions tab ---> Run logs**, check the output of the `Print all variables` step.
    - We should see:

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(369).png)

### What Happened?
- **Workflow-level env vars** apply to all jobs and steps.
- **Job-level env vars** override workflow-level ones for that job.
- **Step-level env vars** override both workflow and job vars for that step.
- **GitHub context variables** (`github.sha`, `github.actor`) provide metadata about the run.

## Task 3: Job Outputs
### File: `.github/workflows/job-outputs.yml`
```yaml
name: Job Outputs Demo

on: [push]

jobs:
  date-job:
    runs-on: ubuntu-latest
    outputs:  #declares outputs that this job will expose to other jobs
      today: ${{ steps.set-date.outputs.date }} #name of the output: pulls value from step with ID 'set-date' and its output named 'date'
    steps:
      - id: set-date  #gives this step an ID so its outputs can be referenced later
        run: echo "date=$(date)" >> $GITHUB_OUTPUT  #writes current date into special '$GITHUB_OUTPUT' file, which registers it as a step output named 'date'

  print-job:
    runs-on: ubuntu-latest
    needs: date-job
    steps:
      - name: Print date from previous job
        run: echo "Today's date is ${{ needs.date-job.outputs.today }}" #prints output 'today' from the 'date-job'. 'needs.date-job.outputs.today' - accesses output named 'today' from  job 'date-job'.
```
### Verification
- Commit and push this workflow to our repo.
- Trigger a run (e.g., by pushing a change).
- In the **Actions tab → Run logs**, check the `print-job` output.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(373).png)

  - We should see:
    
  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(374).png)

### Why would we pass outputs between jobs?
- **Outputs** allow jobs to share data across different runners.
- Each job runs in isolation on a fresh VM, so environment variables or files don’t persist between jobs.
- By setting outputs (`echo "name=value" >> $GITHUB_OUTPUT`), we can pass values like build numbers, artifact paths, or timestamps.
- Why pass outputs between jobs?
    - To reuse computed values (e.g., version numbers, dates, artifact locations).
    - To avoid duplicating logic across jobs.
    - To make downstream jobs dynamic based on upstream results.

## Task 4: Conditionals
### File: `.github/workflows/conditionals.yml`
```yaml
name: Conditionals Demo

on: 
  push: #runs whenever code is pushed to repo
  pull_request: #runs whenever a pull request is opened, updated, or synchronized

jobs:
  conditional-job:
    runs-on: ubuntu-latest
    steps:
      - name: Run only on main branch
        if: github.ref == 'refs/heads/main' #conditional: this step runs only if branch is 'main'.
        run: echo "This runs only on the main branch"

      - name: Step that fails
        run: exit 1 #forces step to fail by exiting with status code '1'.

      - name: Run only if previous step failed
        if: failure() #conditional: this step runs only if a previous step in job failed.
        run: echo "This runs because the previous step failed"

      - name: Continue on error example
        continue-on-error: true #allows this step to fail without failing entire job.
        run: exit 1  #forces step to fail, but because of 'continue-on-error', job continues to next step.

  push-only-job:
    if: github.event_name == 'push' #conditional: this job runs only when workflow was triggered by a push event, not a pull request.
    runs-on: ubuntu-latest
    steps:
      - run: echo "This job runs only on push events, not on pull requests"
```
### Verification
- Push this workflow to our repo.
- Trigger runs on both `push` and `pull_request` events.
- Observe:
  - The main branch step runs only when the branch is main.
  - The failure step runs only if the previous step fails.
  - The push-only job runs only on push events.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(376).png)

  - The continue-on-error step fails but does not stop the job — later steps still execute.

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(377).png)

### What Happened?
- **Branch condition:** `if: github.ref == 'refs/heads/main'` ensures steps run only on `main`.
- **Failure condition:** `if: failure()` lets us react to failed steps.
- **Job condition:** `if: github.event_name == 'push'` restricts jobs to specific event types.
- **continue-on-error:** Allows a step to fail without failing the entire job — useful for non-critical checks (like lint warnings).

## Task 5: Putting It Together
### File: `.github/workflows/smart-pipeline.yml`
```yaml
name: Smart Pipeline

on:
  push:
    branches: ["*"] #matches all branches, so this workflow runs on pushes to any branch.

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Lint code
        run: echo "Linting code..."

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run tests
        run: echo "Running tests..."

  summary:
    runs-on: ubuntu-latest
    needs: [lint, test]   # waits for both jobs
    steps:
      - name: Print summary
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then #checks if branch is 'main'
            echo "Main branch push" #prints message if branch is 'main'
          else
            echo "Feature branch push"  #prints message if it’s any other branch.
          fi
          echo "Commit message: ${{ github.event.commits[0].message }}" #prints commit message from first commit in push event.
```
### Verification
- Push this workflow to our repo.
- Trigger a run by committing to any branch.
- In the **Actions tab → Workflow graph**, we’ll see:
    - **Lint** and **Test** jobs running in parallel.
    - **Summary** job waiting until both complete.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(378).png)

- In the logs for the summary job, we’ll see whether it was a main branch or feature branch push, plus the commit message.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/4e72676bb00c6255ae0693dbf1393eff521cf2d2/2026/day-43/Screenshots/Screenshot%20(380).png)

### What Happened?
- **Parallel jobs:** By default, jobs run independently unless chained with `needs`.
- **Summary job:** Uses `needs: [lint, test]` to wait for both jobs.
- **Branch detection:** `github.ref` lets us distinguish between main and feature branches.
- **Commit message:** Accessed via `github.event.commits[0].message`.
