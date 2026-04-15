# GitHub Actions - Secrets, Artifacts & Running Real Tests in CI
## Task 1: GitHub Secrets
### Creating and Using a Secret
- In our repo, go to **Settings --> Secrets and Variables --> Actions**.
- Click **New repository secret**.
- Name it: `MY_SECRET_MESSAGE`.
- Add any value (e.g., `HelloWorldSecret`).
- Save.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(382).png)

### Workflow Example
```yaml
name: secrets-task
on: [push]

jobs:
  check-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Print secret existence
        run: 'echo "The secret is set: true"' #this is a safe way to confirm that secret exists without revealing its actual value

      - name: Try printing secret directly
        run: 'echo "Secret value: ${{ secrets.MY_SECRET_MESSAGE }}"' #'${{ secrets.MY_SECRET_MESSAGE }}' reference secret stored in our repo’s settings.
```
- The first step prints a safe confirmation: `The secret is set: true`
- The second step tries to print the secret directly. GitHub Actions **masks** the value automatically, so we’ll see `***` in the logs instead of the actual secret.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(385).png)

### Why should we never print secrets in CI logs?
- Logs are often shared or stored permanently.
- Anyone with access to logs could misuse exposed credentials.
- Secrets should only be used as environment variables or inputs, never revealed.

## Task 2: Use Secrets as Environment Variables
### Add Secrets in GitHub
- Go to our repo --> **Settings --> Secrets and Variables --> Actions**.
- Click **New repository secret**.
- Add:
    - `MY_SECRET_MESSAGE` (already created in Task 1).
    - `DOCKER_USERNAME` (Docker Hub username).
    - `DOCKER_TOKEN` (Docker Hub access token or password).

  ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(386).png)

### Create a Workflow File
- Inside `.github/workflows/secrets-env.yml` add:
    ```yaml
    name: secrets-env-demo
    on: [push]

    jobs:
      use-secrets:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v4     #'actions/checkout' pulls our repository code into the runner so subsequent steps can access it

          - name: Use MY_SECRET_MESSAGE as env variable
            env:       #defines environment variables for this step
              SECRET_MSG: ${{ secrets.MY_SECRET_MESSAGE }}    #'SECRET_MSG' is set to value of our GitHub secret 'MY_SECRET_MESSAGE'
            run: |
              echo "The secret is set: true"
              echo "Length of secret is ${#SECRET_MSG}"       #prints length of secret string, not actual value, this is a safe way to verify secret is present without exposing it

          - name: Docker login using secrets
            env:
              DOCKER_USER: ${{ secrets.DOCKER_USERNAME }}
              DOCKER_PASS: ${{ secrets.DOCKER_TOKEN }}
            run: |
              echo "Logging in to Docker Hub..."
              echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin       #securely logs in to Docker Hub using the secrets, password is piped in via '--password-stdin' so it doesn’t appear in logs.
    ```

- Save the file.
- Commit and push.

### Observe the Run
- Go to **Actions tab** in GitHub.
- Open the workflow run.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(388).png)

- We’ll see:
    - `The secret is set: true`
    - `Length of secret is …` (but not the actual secret).

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(389).png)

    - Docker login step succeeds if our credentials are correct.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(390).png)

### What Happened?
- Secrets can be passed as environment variables safely.
- GitHub masks secret values (`***`) in logs automatically.
- Never hardcode credentials in workflows - always use secrets.
- Docker credentials are stored securely and will be used in Day 45.

## Task 3: Upload Artifacts
### Create a Workflow File
- Inside our repo, add a new workflow file: `.github/workflows/upload-artifact.yml`
    ```yaml
    name: upload-artifact-demo
    on: [push]

    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v4   #pulls our repository code into runner so subsequent steps can access it - by default, runner starts with a clean environment and does not have our repository’s code

          - name: Generate a test report
            run: echo "This is a sample test report for Day 44" > report.txt

          - name: Upload artifact
            uses: actions/upload-artifact@v4 #saves files from runner so we can download them later
            with:
              name: test-report   #artifact will be labeled “test-report” in  Actions tab
              path: report.txt    #file to upload
    ```
### Run the Workflow
- Go to our repo’s **Actions tab**.
- Select the workflow run.
- We’ll see a job called **build**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(414).png)

### Download the Artifact
- Scroll down to the **Artifacts** section in the workflow run summary.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(415).png)

- We should see `test-report`.
- Click it to download the `.zip` file.
- Extract it locally --> we’ll find `report.txt` with the content we generated.
### What Happened?
- Artifacts are files generated during a workflow (e.g., logs, reports, binaries).
- They can be uploaded with `actions/upload-artifact@v4`.
- After the run, artifacts are available in the **Actions tab** for download.
- This is useful for debugging, sharing test reports, or passing build outputs between jobs.

## Task 4: Download Artifacts Between Jobs
### Create Workflow File
- Add a new workflow file: `.github/workflows/artifact-between-jobs.yml`
    ```yaml
    name: artifact-between-jobs
    on: [push]

    jobs:
      job1:
        runs-on: ubuntu-latest
        steps:
          - name: Generate file
            run: echo "This file was created in Job 1" > shared.txt

          - name: Upload artifact
            uses: actions/upload-artifact@v4    #saves file to GitHub’s servers so other jobs can download it
            with:
              name: shared-file
              path: shared.txt

      job2:
        runs-on: ubuntu-latest
        needs: job1       #ensures job2 only runs after job1 completes successfully
        steps:
          - name: Download artifact
            uses: actions/download-artifact@v4    #downloads artifact named shared-file into workspace
            with:
              name: shared-file

          - name: Print file contents
            run: cat shared.txt   #prints contents of 'shared.txt' to workflow logs.
    ```
### Observe Workflow Run
- Go to the **Actions tab**.
- We’ll see two jobs: **job1** and **job2**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(416).png)

- **job1** generates and uploads `shared.txt`.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(417).png)

- **job2** downloads it and prints the contents.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(418).png)

### When would we use artifacts in a real pipeline?
- **Artifacts between jobs** allow us to pass files from one job to another.
- In real pipelines, artifacts are used for:
    - Sharing build outputs (e.g., compiled binaries).
    - Passing test reports or logs to later jobs.
    - Delivering packaged files to deployment stages.
- This ensures jobs remain isolated but can still exchange necessary data.

## Task 5: Run Real Tests in CI
### Add a Script to Your Repo
Save below script as `fetch_api.py` in the root of our repo. This script is from my `python-for-devops` repo.
```python
import requests #Loads the Requests library to make HTTP calls (GET/POST).
import json #Provides tools to convert Python data to/from JSON.

def fetch_api_data():
    try:
        response = requests.get("https://jsonplaceholder.typicode.com/users")
        response.raise_for_status()
    except Exception as e:
        print("API request failed: ",e)
        return
    
    try:
        users = response.json()
    except Exception as e:
        print("Error reading JSON: ",e)
        return

    processed_data = []
    for user in users[:5]:
        processed_data.append({
            "id": user["id"],
            "name" : user["name"],
            "email" : user["email"],
            "city" : user["address"]["city"],
            "company" : user["company"]["name"]
        })

    print("---Processed API Data---")
    for item in processed_data:
        print(f"ID: {item['id']} | Name: {item['name']} | Email: {item['email']} | City: {item['city']} | Company: {item['company']}")

    try:
        with open("output.json", "w") as f:
            json.dump(processed_data, f, indent=4)
        print("nData saved to output.json")
    except Exception as e:
        print("Error saving file: ",e)

fetch_api_data()
```
Since our script uses `requests`, add a `requirements.txt` file:
```Code
requests
```
### Create Workflow File
Add `.github/workflows/run-fetch-api.yml`:
```yaml
name: run-fetch-api-demo
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4   #pulls our repository code into runner, without this runner wouldn’t have access to our 'fetch_api.py' or 'requirements.txt'

      - name: Setup Python
        uses: actions/setup-python@v5   #installs Python in runner
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt    #uses 'pip' to install packages listed in 'requirements.txt'

      - name: Run fetch_api script
        run: python fetch_api.py    #executes 'fetch_api.py', if the script exits with code (0), the job passes (green)
```
### Push & Observe Workflow Run
- Go to **Actions tab**.
- The script will:
  - Fetch API data.
  - Print processed results.
  - Save `output.json`.
    - When the workflow runs, the script executes inside the ephemeral runner’s workspace. `output.json` will be saved there temporarily.
    - Unless we upload it as an artifact, it will be discarded when the runner is destroyed at the end of the job.
    - To make sure we can download it after the workflow finishes, add an artifact upload step at the end:
      ```yaml
        - name: Upload output.json
          uses: actions/upload-artifact@v4
          with:
            name: api-output
            path: output.json
      ```
- If everything works, pipeline goes **green**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(419).png)

### Intentionally Break the Script
Edit `fetch_api.py` to force an error:

```python
import requests   # Loads the Requests library to make HTTP calls
import json       # Provides tools to convert Python data to/from JSON
import sys        # Needed to exit with non-zero codes

def fetch_api_data():
    try:
        response = requests.get("https://invalid-url.typicode.com/users")
        response.raise_for_status()
    except Exception as e:
        print("API request failed:", e)
        sys.exit(1)   # Fail the pipeline

    try:
        users = response.json()
    except Exception as e:
        print("Error reading JSON:", e)
        sys.exit(1)   # Fail the pipeline

    processed_data = []
    for user in users[:5]:
        processed_data.append({
            "id": user["id"],
            "name": user["name"],
            "email": user["email"],
            "city": user["address"]["city"],
            "company": user["company"]["name"]
        })

    print("---Processed API Data---")
    for item in processed_data:
        print(f"ID: {item['id']} | Name: {item['name']} | Email: {item['email']} | City: {item['city']} | Company: {item['company']}")

    try:
        with open("output.json", "w") as f:
            json.dump(processed_data, f, indent=4)
        print("\nData saved to output.json")
    except Exception as e:
        print("Error saving file:", e)
        sys.exit(1)   # Fail the pipeline

fetch_api_data()

```
Commit and push:

```bash
git commit -am "Break fetch_api.py intentionally"
git push origin main
```
Workflow fails --> pipeline goes **red**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(408).png)

### Fix the Script
Restore the correct `fetch_api.py` then
Commit and push:
```bash
git commit -am "Fix fetch_api.py"
git push origin main
```
Workflow passes again --> pipeline goes **green**.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(405).png)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(406).png)

## Task 6: Caching
### Create Workflow File
- Add a new workflow file: `.github/workflows/cache-demo.yml`
    ```yaml
    name: cache-demo
    on: [push]

    jobs:
      cache-example:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v4   #pulls our repository code into runner

          - name: Cache pip dependencies
            uses: actions/cache@v4    #saves files between workflow runs
            with:
              path: ~/.cache/pip    #directory where pip stores downloaded packages
              key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}     #unique identifier for cache, it combines runner OS and a hash of 'requirements.txt'. If 'requirements.txt' changes, the hash changes ---> cache is invalidated.
              restore-keys: |
                ${{ runner.os }}-pip-   #fallback key, if exact key isn’t found - GitHub tries to restore a cache with a prefix match (e.g., ubuntu-latest-pip-).

          - name: Setup Python
            uses: actions/setup-python@v5   #installs Python 3.11 in the runner
            with:
              python-version: '3.11'

          - name: Install dependencies
            run: |
              if [ -f requirements.txt ]; then pip install -r requirements.txt; fi     #installs dependencies from 'requirements.txt'. On first run, pip downloads packages from PyPI and stores them in '~/.cache/pip'. On subsequent runs, cache restores those packages ---> installation is much faster.

          - name: Run script
            run: python hello.py    #runs our Python script, this is actual workload after dependencies are ready.
    ```
### Observe Workflow Runs
- **First run:** Dependencies are installed fresh --> takes longer.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(411).png)

- **Second run:** Cache is restored --> dependency installation is much faster.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(412).png)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/34943196240c7adae0593a4a6b9ba4cde7b0bb39/2026/day-44/Screenshots/Screenshot%20(413).png)

### What is being cached and where is it stored?
**What is cached?** Dependency files (e.g., Python packages in `~/.cache/pip`).

**Where is it stored?** On GitHub’s servers, tied to our workflow cache key.

**Why use caching?** Speeds up builds by avoiding repeated downloads/installs.
