# GitHub Actions - Reusable Workflows & Composite Actions
## Task 1: Understanding `workflow_call`
### What is a reusable workflow?
**Ans.** A **reusable workflow** is a GitHub Actions workflow file that is designed to be *called* by other workflows rather than triggered by repo events directly. Instead of copy-pasting the same build/test/deploy logic across many repos, we define it once and reference it like a function.
 
Reusable workflows can accept **inputs** (typed parameters), **secrets**, and return **outputs** - making them first-class building blocks in a CI/CD system.    
### What is the `workflow_call` trigger?
```yaml
on: 
  workflow_call:                                                #special trigger used for reusable workflows, it allows other workflows to invoke this one and pass in inputs, secrets, and consume outputs
    inputs:                                                     #declare parameters that caller workflow must (or can) provide
      app_name:
        type: string                                            #value must be a string
        required: true                                          #caller workflow must supply this input, otherwise reusable workflow won’t run
    secrets:                                                    #declare secrets that must be passed in from caller workflow, stored in repo’s Settings --> Secrets.
      docker_token:
        required: true                                          #caller workflow must provide this secret when invoking reusable workflow
    outputs:                                                    #declare values that this reusable workflow will expose back to caller workflow
      build_version:
        value: ${{ jobs.build.outputs.build_version }}          #maps workflow’s output 'build_version' to output of 'build' job
                                                                #inside 'jobs.build' definition, we’ll have a step that sets 'build_version' (e.g., 'echo "build_version=v1.0-${SHORT_SHA}" >> $GITHUB_OUTPUT')- that step’s output is then surfaced here so caller workflow can read it.
```
 
**Ans.** `workflow_call` is a special event trigger that marks a workflow as *callable*. When this trigger is present, the workflow **will not run on its own** from push/PR events - it waits to be invoked by a caller workflow using `uses:`.
 
It supports three declaration blocks:
- `inputs:` — typed parameters (`string`, `boolean`, `number`) with optional defaults
- `secrets:` — secrets passed from the caller's context
- `outputs:` — values the reusable workflow exposes back to the caller

### How is calling a reusable workflow different from using a regular action (`uses:`)?
| Aspect | Reusable Workflow (`workflow_call`) | Regular Action (`uses:` in a step) |
|---|---|---|
| **Unit of reuse** | Entire workflow with jobs | A single step |
| **Can contain jobs** | ✅ Yes, multiple jobs | ❌ No — is itself a step |
| **Trigger mechanism** | `uses:` at the **job** level | `uses:` at the **step** level |
| **Accepts secrets** | ✅ Via `secrets:` block | ✅ Via `with:` (env vars) |
| **Has its own runner** | ✅ Each job gets its own | ❌ Runs on parent job's runner |
| **Example** | `uses: ./.github/workflows/build.yml` | `uses: actions/checkout@v4` |
 
**Key distinction**: calling a reusable workflow happens at the **`jobs:` level**, not inside a job's `steps:`. We're invoking a full workflow, not a single action step.
### Where must a reusable workflow file live?
- **Same repo**: `.github/workflows/` — called with `uses: ./.github/workflows/file.yml`
- **Another repo**: `.github/workflows/` in that repo — called with
  `uses: org/repo/.github/workflows/file.yml@ref`
- The repo must be **public**, or the caller must have access (same org with settings enabled)
- The `@ref` (branch, tag, or SHA) is **required** for cross-repo calls

## Task 2: Create Our First Reusable Workflow
Create `.github/workflows/reusable-build.yml`:
```yaml
name: Reusable Build Workflow #human‑readable label, it shows up in Actions tab when workflow is called by another workflow

on:
  workflow_call:                                                                     #this workflow is triggered only when another workflow explicitly calls it
    inputs:                                                                          #defines parameters caller workflow must provide
      app_name:                                                                      #required string input, caller must pass application name
        description: "Name of the application"
        required: true
        type: string
      environment:                                                                   #string input, it’s required, but has a default value of 'staging'
        description: "Deployment environment"
        required: true
        type: string
        default: staging
    secrets:                                                                         #declares sensitive values caller must provide
      docker_token:                                                                  #required secret, caller workflow must pass in a Docker registry token
        description: "Docker registry token"
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4                                                    #checks out our repo code so workflow has access to it

      - name: Print build info                                                       #prints a message using inputs passed in by caller workflow
        run: echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Verify Docker token                                                    #verifies whether secret 'docker_token' was passed
        run: |
          if [ -n "${{ secrets.docker_token }}" ]; then                              #shell 'if' checks if it’s non‑empty (-n)
            echo "Docker token is set: true"
          else
            echo "Docker token is set: false"
          fi
```
- Push the file to our repo
- The **Reusable Workflow** won’t show up in the Actions tab by itself because it’s defined with `on: workflow_call`. Reusable workflows are like “functions”: they only run when another workflow **calls them**.

## Task 3: Create a Caller Workflow
### Create `.github/workflows/call_build.yml`:
```yaml
name: Caller Workflow

on:
  push:                                                         #unlike reusable workflow, this one has a normal trigger so it shows up in Actions tab
    branches:
      - main

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml                #this job calls our reusable workflow located at '.github/workflows/reusable-build.yml'
    with:                                                       #inputs passed to reusable workflow
      app_name: "my-web-app"
      environment: "production"
    secrets:                                                    #secrets passed to reusable workflow
      docker_token: ${{ secrets.DOCKER_TOKEN }}
```
### Add the secret in GitHub
- Go to our repo **--> Settings --> Secrets and variables --> Actions**.
- Add a new secret named `DOCKER_TOKEN` with our Docker registry token value.
- This ensures the caller workflow can pass the secret to the reusable workflow.
- Save the file.
- Run:
  ```bash
  git add .github/workflows/call-build.yml
  git commit -m "Add caller workflow to trigger reusable build"
  git push origin main
  ```
### Verify in Actions tab
- Go to our repo’s **Actions** tab.
- We should see the **Caller Workflow** triggered by our push.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(459).png)

- Click into the job --> we’ll see:
  - `Building my-web-app for production`

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(457).png)

  - `Docker token is set: true`

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(467).png)

## Task 4: Add Outputs to the Reusable Workflow
### Update reusable workflow
Edit `.github/workflows/reusable-build.yml`.
```yaml
name: Reusable Build Workflow                                                   #appears in Actions tab when invoked by a caller workflow

on:
  workflow_call:                                                                #'workflow_call' makes this workflow reusable
    inputs:                                                                     #parameters caller workflow must provide
      app_name:
        description: "Name of the application"
        required: true
        type: string
      environment:
        description: "Deployment environment"
        required: true
        type: string
        default: staging
    secrets:
      docker_token:
        description: "Docker registry token"
        required: true
    outputs:                                                                     #values this workflow exposes back to caller
      build_version:                                                             #maps to 'build_version' output of the 'build' job
        description: "Generated build version"
        value: ${{ jobs.build.outputs.build_version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:                                                                     #declares a job output 'build_version', which comes from step with 'id: version'
      build_version: ${{ steps.version.outputs.build_version }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print build info
        run: echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Verify Docker token
        run: |
          if [ -n "${{ secrets.docker_token }}" ]; then
            echo "Docker token is set: true"
          else
            echo "Docker token is set: false"
          fi

      - name: Generate build version                                              #generates a build version string using short commit SHA
        id: version                                                               #allows referencing this step’s outputs, later we reference 'steps.version.outputs.build_version' when defining job outputs - without an id we can’t access the step’s outputs
        run: |
          SHORT_SHA=$(echo "${GITHUB_SHA}" | cut -c1-7)                           #takes full commit SHA (${GITHUB_SHA}) & slice the first 7 characters - Stores it in a variable 'SHORT_SHA'
          echo "build_version=v1.0-${SHORT_SHA}" >> $GITHUB_OUTPUT
                                                                                  # '$GITHUB_OUTPUT' is how we set outputs from a step in GitHub Actions. Here, the output name is 'build_version', and the value is 'v1.0-<short-sha>'. Because step has 'id: version', this output is now available as 'steps.version.outputs.build_version'.
```
### Update caller workflow
Edit `.github/workflows/call-build.yml`.
```yaml
name: Caller Workflow

on:
  push:
    branches:
      - main

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml                                                    #instead of steps, it calls our reusable workflow located at '.github/workflows/reusable-build.yml'
    with:                                                                                           #passes inputs into reusable workflow
      app_name: "my-web-app"
      environment: "production"
    secrets:                                                                                        #passes required secret into reusable workflow
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  print-version:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Print build version                                                                   #prints 'build_version' output that was generated inside reusable workflow (Generate build version step).
        run: echo "Build version from reusable workflow: ${{ needs.build.outputs.build_version }}"  #how our 'caller workflow' accesses output from reusable workflow’s 'build' job

        #${{ ... }} → GitHub Actions expression syntax. Anything inside is evaluated at runtime
        #'needs.build' → Refers to job named 'build' in our caller workflow, because our 'print-version' job has 'needs: build', it can access outputs from that job
        #'outputs.build_version' → Refers to specific output we exposed in reusable workflow
            #Inside our reusable workflow, the 'Generate build version' step set 'build_version' as a step output
            #That was mapped to job output (jobs.build.outputs.build_version)
            #Then mapped again to workflow output (workflow_call.outputs.build_version)
            #Finally, caller workflow can consume it here
```
### Commit and push
```bash
git add .github/workflows/reusable-build.yml .github/workflows/call-build.yml
git commit -m "Add outputs to reusable workflow and consume in caller"
git push origin main
```
### Verify in Actions tab
- Push a commit to `main`.
- In the Actions tab, open the workflow run.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(461).png)

- We should see:
  - `Building my-web-app for production`
  - `Docker token is set: true`

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(463).png)

  - `Build version from reusable workflow: v1.0-<short-sha>`

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(462).png)

## Task 5: Create a Composite Action
### Create a action file `.github/actions/setup-and-greet/action.yml`
```yaml
name: "Setup and Greet"
description: "Custom composite action to greet user"          #short explanation of what action does

inputs:                                                       #defines parameters action accepts
  name:                                                       #required input, Caller must provide a name
    description: "Name to greet"
    required: true
  language:                                                   #Optional input, defaults to "en" - Caller can override with "es" or "fr"
    description: "Language for greeting"
    required: false
    default: "en"

outputs:                                                      #defines values action exposes back to workflow
  greeted:                                                    #output flag
    description: "Whether greeting was printed"
    value: ${{ steps.greet.outputs.greeted }}                 #maps to output set by step with 'id: greet'

runs:                                                         #defines how action executes
  using: "composite"                                          #this action is a composite action (runs multiple steps defined here)
  steps:                                                      #list of steps that make up action
    - name: Print greeting
      id: greet                                               #identifier so its outputs can be referenced
      run: |
        if [ "${{ inputs.language }}" = "en" ]; then          #conditional logic → chooses greeting based on 'inputs.language'
          echo "Hello, ${{ inputs.name }}!"
        elif [ "${{ inputs.language }}" = "es" ]; then
          echo "¡Hola, ${{ inputs.name }}!"
        elif [ "${{ inputs.language }}" = "fr" ]; then
          echo "Bonjour, ${{ inputs.name }}!"
        else
          echo "Hello, ${{ inputs.name }}!"
        fi
        echo "greeted=true" >> $GITHUB_OUTPUT                 #sets step output 'greeted=true', this is what connects to outputs block above
      shell: bash                                             #runs script in Bash

    - name: Print date and OS                                 #useful for debugging and confirming environment details
      run: |
        echo "Current date: $(date)"
        echo "Runner OS: $RUNNER_OS"
      shell: bash
```
### Create the workflow file
File: `.github/workflows/use-greet.yml`
```yaml
name: Use Composite Action

on:
  push:
    branches:
      - main

jobs:
  greet-job:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run setup-and-greet 
        id: greet                                                               #assigns an identifier to this step so its outputs can be referenced later
        uses: ./.github/actions/setup-and-greet                                 #points to folder containing 'action.yml'
        with:                                                                   #passes inputs into action
          name: "Atul"                                                          #required input
          language: "en"

      - name: Verify output                                                     #prints output from composite action
        run: echo "Greeting completed: ${{ steps.greet.outputs.greeted }}"      #${{ steps.greet.outputs.greeted }} → references output greeted from step with 'id: greet'
```
### Commit and push
```bash
git add .github/actions/setup-and-greet/action.yml .github/workflows/use-greet.yml
git commit -m "Add composite action and workflow to greet"
git push origin main
```
### Verify in Actions tab
- Push a commit to `main`.
- In the Actions tab, open the run.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(464).png)

- We should see:
  - A greeting in the specified language (e.g., `Hello, Atul!`).
  - Current date and runner OS printed.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/93db23de4dff68699ce55124e1ab71b227c16ac5/2026/day-46/Screenshots/Screenshot%20(465).png)

  - Output `greeted=true`.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1beb578830f7fb988b5b91b2862f709387f421f0/2026/day-46/Screenshots/Screenshot%20(469).png)

## Task 6: Reusable Workflow vs Composite Action
| Feature | Reusable Workflow | Composite Action |
|---|---|---|
| **Triggered by** | `workflow_call` event | `uses:` inside a **step** |
| **Can contain jobs** | ✅ Yes, multiple parallel jobs | ❌ No — contains steps only |
| **Can contain multiple steps** | ✅ Yes (within each job) | ✅ Yes |
| **Lives where** | `.github/workflows/` | `.github/actions/<name>/action.yml` |
| **Can accept secrets directly** | ✅ Yes — via `secrets:` block | ⚠️ No — pass via `env:` or `with:` |
| **Has own runner** | ✅ Each job gets its own | ❌ Shares caller job's runner |
| **Supports matrix strategy** | ✅ Yes | ❌ No |
| **Cross-repo reuse** | ✅ Yes (`org/repo/.github/workflows/file@ref`) | ✅ Yes (`org/repo/.github/actions/name@ref`) |
| **Best for** | Multi-job pipelines, full CI workflows, anything needing parallelism or separate environments | Encapsulating reusable step logic, setup tasks, utility helpers |
 
### Decision Guide
 
```
Need multiple jobs or parallelism?          → Reusable Workflow
Need to encapsulate a few steps?            → Composite Action
Want to accept secrets cleanly?             → Reusable Workflow
Want to run inline in an existing job?      → Composite Action
Cross-repo standard CI pipeline?            → Reusable Workflow
Cross-repo utility (lint setup, greet)?     → Composite Action
```

## Key Takeaways
 
1. **`workflow_call` = function signature** for a workflow. Declare inputs, secrets, outputs.
2. **Caller syntax is at job level** (`uses:` under `jobs:`, not `steps:`).
3. **Outputs need 3-level wiring**: step → job → workflow_call → caller's `needs.job.outputs`.
4. **Composite actions are step-level reuse**: perfect for encapsulating setup logic.
5. **Composite actions require `shell: bash`** (or another shell) on every step — unlike regular
   workflow steps, the shell is not assumed.
6. **Local composite actions need `actions/checkout` first** so the runner can find the file.
7. **Secrets cannot flow through composite action `with:` inputs directly** — use `secrets:` in reusable workflows when secret passing is needed.
