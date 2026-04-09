# GitHub Actions – Triggers & Matrix Builds

## Task 1: Trigger on Pull Request
### Create the workflow file
- In our repo, add a new file at `.github/workflows/pr-check.yml`:
    ```yaml
    name: PR Check    # workflow name, just a label

    on:               # defines when workflow should run
      pull_request:   # runs whenever a pull request is opened
        branches: [main]  # limits trigger to PRs targeting main branch only.

    jobs:             # workflow is made of jobs. each job runs independently.
      pr-check:       # job’s ID (we can name it anything)
        runs-on: ubuntu-latest  # Specifies runner environment
        steps:        # job is made of steps, executed sequentially.
          - name: Print branch name # label for step (human-readable)
            run: echo "PR check running for branch: ${{ github.head_ref }}" # executes shell command on runner.
            # ${{ github.head_ref }}: A GitHub Actions context variable. For PRs, it contains the source branch name (e.g., feature/test-pr-trigger).
    ```
- Commit and push it to our repo
    ```bash
    git add .github/workflows/pr-check.yml
    git commit -m "Add PR check workflow"
    git push origin main
    ```
### Push a branch and open a PR
- Create a new branch locally:
    ```bash
    git checkout -b feature/test-pr-trigger
    ```
- Make a small commit:
    ```bash
    echo "test" > test.txt
    git add test.txt
    git commit -m "Test PR trigger"
    git push origin feature/test-pr-trigger
    ```
- Open a Pull Request against `main`.
    - Navigate to our repository on GitHub in the browser.
    - We’ll see a banner suggesting us **Compare & pull request** for the branch we just pushed. Click that button.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/555151b942370fddd57d6758d549dd22d617ade5/2026/day-41/Screenshots/Screenshot%20(315).png)

    - Fill in PR details
        - **Base branch:** `main`
        - **Compare branch:** `feature/test-pr-trigger`
        - Add a title and description.
        - Click **Create pull request**.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/555151b942370fddd57d6758d549dd22d617ade5/2026/day-41/Screenshots/Screenshot%20(316).png)

### Observe the workflow
- As soon as the PR is opened, GitHub Actions will automatically trigger the workflow.
- Go to the **PR page** ---> scroll down to **Checks** ---> we should see **PR Check** listed, confirming that our trigger is working correctly.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(321).png)

- Click into the workflow run ---> check the logs. We’ll see:
    ```Code
    PR check running for branch: feature/test-pr-trigger
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(331).png)

## Task 2: Scheduled Trigger
### Add a Scheduled Trigger
- We can add a `schedule` block to our workflow file. For example:
    ```yaml
    name: Scheduled Workflow

    on:
      schedule:   # tells GitHub to run workflow on a cron schedule.
        - cron: "0 0 * * *"   # every day at midnight UTC

    jobs:
      scheduled-job:      # ID of this job.
        runs-on: ubuntu-latest
        steps:
          - name: Print message
            run: echo "This workflow runs every day at midnight UTC"

    ```
- The cron expression `"0 0 * * *"` means:
    - Minute = 0
    - Hour = 0
    - Every day, every month, every day of week ---> **midnight UTC daily**

### What is the cron expression for every Monday at 9 AM?
```Code
0 9 * * 1
```
- Minute = 0
- Hour = 9
- Day of week = 1 (Monday)

### Verification
- After committing this workflow, go to the **Actions tab**.
- We’ll see the workflow listed, but note: scheduled workflows **don’t run immediately** after creation. They’ll trigger at the next scheduled time.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(324).png)

- We can also manually trigger them once (using `workflow_dispatch`) if you want to test right away.
    ```yaml
    name: Scheduled Workflow

    on:
      schedule:
        - cron: "0 0 * * *"   # every day at midnight UTC
      workflow_dispatch:  # trigger workflow manually from Actions tab, Since no inputs are defined, GitHub won’t ask for any parameters when we run it manually - it just starts immediately.

    jobs:
      scheduled-job:
        runs-on: ubuntu-latest
        steps:
          - name: Print message
            run: echo "This workflow runs every day at midnight UTC"

    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(327).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(330).png)

## Task 3: Manual Trigger
### Create the workflow file
- Add a new file at `.github/workflows/manual.yml`:
    ```yaml
    name: Manual Workflow

    on:
      workflow_dispatch:    # trigger workflow manually from Actions tab
        inputs:             # defines parameters we can pass when running workflow manually.
          environment:      # name of input.
            description: "Choose environment" # text shown in UI to explain input.
            required: true # we must select a value before running
            default: "staging"
            type: choice   # renders a dropdown menu in UI
            options:       # choices available in dropdown (staging or production).
              - staging
              - production

    jobs:
      run-manual:         # ID of this job.
        runs-on: ubuntu-latest  # runner environment, GitHub’s hosted Ubuntu VM.
        steps:
          - name: Print environment
            run: echo "Running in environment: ${{ github.event.inputs.environment }}"
            # ${{ github.event.inputs.environment }}: A GitHub Actions context variable that retrieves the value chosen in the dropdown (staging or production).
    ```
### Trigger the workflow manually
- Push this file to our repo.
- Go to the **Actions** tab in GitHub.
- Find **Manual Workflow** in the list.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(333).png)

- Click **Run workflow**.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(334).png)
    
- Select either **staging** or **production** from the dropdown.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(336).png)

- Confirm and run.

### Verify the output
- Open the workflow run logs.
- We should see a line like:

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(337).png)

depending on our input.

## Task 4: Matrix Builds
### Create the workflow file
- Add a new file at `.github/workflows/matrix.yml`:
    ```yaml
    name: Matrix Build

    on: push   # runs whenever we push commits to the repo

    jobs:
      build:    # ID of this job.
        runs-on: ubuntu-latest
        strategy:   # defines how job should be repeated with different parameters.
          matrix:   # creates multiple job runs, one for each value in matrix.
            python-version: ["3.10", "3.11", "3.12"]  # variable being iterated, here it runs the job three times
        steps:
          - name: Setup Python 
            uses: actions/setup-python@v5 # uses official actions/setup-python action to install specified Python version.
            with:
              python-version: ${{ matrix.python-version }} # ${{ matrix.python-version }}: Pulls current Python version from matrix (3.10, 3.11, or 3.12 depending on run)
          - name: Print Python version
            run: python --version
    ```
    - This will run 3 jobs in parallel, one for each Python version.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(342).png)

### Extend the matrix with operating systems
- Update the strategy block:
    ```yaml
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ["3.10", "3.11", "3.12"]
    ```
- And update the job runner:
    ```yaml
    jobs:
      build:
        runs-on: ${{ matrix.os }} # instead of hardcoding, it uses matrix variable os, that means each job will run on OS defined in matrix (Ubuntu or Windows)
        strategy: # defines how job should be repeated with different parameters
          matrix: # creates multiple job runs for each combination of values
            os: [ubuntu-latest, windows-latest]
            python-version: ["3.10", "3.11", "3.12"]
        steps:
          - name: Setup Python
            uses: actions/setup-python@v5
            with:
              python-version: ${{ matrix.python-version }}
          - name: Print Python version
            run: python --version
    ```
### Verification
- With just Python versions: **3 jobs**.
- With Python versions × 2 OS: **3 × 2 = 6 jobs**.
- We’ll see all jobs run in parallel in the Actions tab.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(344).png)

### How many total jobs run now?
- **Matrix builds** let us test across multiple environments automatically.
- **Initial run:** 3 jobs (Python 3.10, 3.11, 3.12).
- **Extended run:** 6 jobs (3 versions × 2 OS).
- Useful for ensuring compatibility across versions and platforms.

## Task 5: Exclude & Fail-Fast
### Update your matrix workflow
- Extend our `.github/workflows/matrix.yml` to include **exclude** and **fail-fast**:
    ```yaml
    name: Matrix Build with Exclude

    on: push

    jobs:
      build:
        runs-on: ${{ matrix.os }} # uses OS defined in matrix (ubuntu-latest or windows-latest)
        strategy:
          fail-fast: false  # ensures that if one job fails, others continue running
          matrix:   # creates combinations of OS × Python version
            os: [ubuntu-latest, windows-latest]
            python-version: ["3.10", "3.11", "3.12"]
            exclude:    # removes one combination (Windows + Python 3.10)
              - os: windows-latest
                python-version: "3.10"
        steps:
          - name: Setup Python
            uses: actions/setup-python@v5
            with:
              python-version: ${{ matrix.python-version }}  # pulls current Python version for jo
          - name: Print Python version
            run: python --version
          - name: Force failure (for testing)
            if: ${{ matrix.python-version == '3.11' && matrix.os == 'ubuntu-latest' }}  # conditional step, runs only if job is Ubuntu + Python 3.11
            run: exit 1 # forces job to fail, this is useful for testing how failures are reported in matrix
    ```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8bba0d1604323612754833f2fce41c90ad59e698/2026/day-41/Screenshots/Screenshot%20(346).png)

- Exclude: The combination `windows-latest + Python 3.10` will **not** run.
- **Fail-fast: false:** Even if one job fails (like the forced failure above), the other jobs continue running.
- We’ll see multiple jobs complete successfully while one fails.

### What does fail-fast: true (the default) do vs false?
- **fail-fast: true (default):**
    - If any job in the matrix fails, GitHub cancels all remaining jobs immediately.
    - Useful when we want to stop wasting resources once a failure is detected.
- **fail-fast: false:**
    - Other jobs continue running even if one fails.
    - Useful when we want to see results across all environments, regardless of failures.
### Verification
Push this workflow, watch the jobs run in parallel, confirm that the excluded combination doesn’t appear, and observe that other jobs continue even after one fails.

## Key Takeaways
 
- **Multiple triggers** can coexist in a single `on:` block.
- **`pull_request`** events expose `github.head_ref` (source branch) and `github.base_ref` (target branch).
- **Cron** runs are always in UTC. Monday at 9 AM = `0 9 * * 1`.
- **Matrix strategy** multiplies combinations — 3 versions × 2 OS = 6 jobs, all parallel.
- **`exclude`** removes specific combinations from the matrix.
- **`fail-fast: false`** lets all matrix jobs complete even when one fails — critical for debugging cross-platform issues.