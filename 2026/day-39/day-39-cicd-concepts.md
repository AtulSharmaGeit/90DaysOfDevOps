# What is CI/CD?

## Task 1: The Problem

### Scenario: A team of 5 developers all pushing code to the same repo manually deploying to production.
1. What can go wrong?<br>
**Ans.**
    - **Conflicting changes:** Two developers push code that overwrites or breaks each other’s work.
    - **Unstable builds:** Code may compile locally but fail in production.
    - **Human error:** Manual deployment steps are prone to mistakes (forgetting a command, wrong server, missing dependency).
    - **No consistency:** Each developer may follow slightly different steps, leading to unpredictable results.
    - **Delayed feedback:** Bugs are discovered late, often in production, instead of during development.

2. What does "it works on my machine" mean and why is it a real problem?<br>
**Ans.**
    - Developers often configure their local environment differently (different OS, versions, libraries, environment variables).
    - Code that runs fine locally may fail in staging or production because those environments don’t match the developer’s setup.
    - This phrase highlights the **lack of reproducibility** - a major barrier to team collaboration and reliable deployments.

3. How many times a day can a team safely deploy manually?<br>
**Ans.**
    - Realistically **1–2 times per day** at most.
    - Beyond that, the risk of downtime, conflicts, and errors grows too high.
    - Frequent manual deployments are unsustainable, especially with multiple developers pushing changes.

## Task 2: CI vs CD
### Continuous Integration (CI)
1. **Definition:** Developers frequently merge code into a shared repository. Automated builds and tests run on each commit or pull request.
2. **Purpose:** Catches integration issues, broken builds, and failing tests early.
3. **Frequency:** Happens multiple times a day, ideally on every commit.
4. **Example:** A developer opens a pull request on GitHub. Within two minutes, GitHub Actions runs 400 unit tests and a linter. The PR is blocked from merging until all checks pass. The team never merges broken code.

### Continuous Delivery (CD)
1. **Definition:** Builds on CI by ensuring the application is always in a deployable state. Code is packaged, tested, and ready for release at any time.
2. **Difference from CI:** CI focuses on integration/testing; CD focuses on preparing code for release. Deployment may still be a manual decision.
3. **Example:** After CI passes, the pipeline automatically deploys to a staging server and runs smoke tests. The product manager reviews it there and clicks "Deploy to Production" in Slack. Five seconds later it's live.

### Continuous Deployment (CD)
1. **Definition:** Extends Continuous Delivery by automatically deploying every successful build/test to production without human intervention.
2. **Difference from Delivery:** Delivery stops at “ready to release”; Deployment actually pushes changes live.
3. **When used:** Teams with strong automated testing and confidence in their pipelines (e.g., SaaS companies needing rapid feature rollout).
4. **Example:** A developer merges a bug fix at 3pm. By 3:02pm it is live in production, having passed unit tests, integration tests, a canary deploy to 1% of traffic, and a health check — all without anyone clicking anything.

## Task 3: Pipeline Anatomy
| Component | What it does |
|-----------|-------------|
| **Trigger** | The event that kicks off the pipeline. Usually a `git push`, a pull request being opened, a tag being created, or a scheduled cron job. The pipeline sits dormant until a trigger fires. |
| **Stage** | A logical phase that groups related work. Common stages: `build`, `test`, `deploy`. Stages run in sequence — if `test` fails, `deploy` never runs. They're the high-level chapters of the pipeline story. |
| **Job** | A discrete unit of work that runs inside a stage. A `test` stage might have separate jobs for unit tests, integration tests, and linting — these can run in parallel. Each job starts fresh. |
| **Step** | A single command or action inside a job. Steps run sequentially within a job. One step checks out the code, the next installs dependencies, the next runs the test command. |
| **Runner** | The machine (virtual or physical) that actually executes a job. GitHub Actions runners are ephemeral VMs — spun up for one job, then destroyed. You can also self-host runners on your own infrastructure. |
| **Artifact** | A file or directory produced by a job and made available to later stages. A compiled binary, a Docker image, a test coverage report, a build folder. Without explicit artifact storage, outputs from one job don't survive to the next. |

## Task 4: Draw a Pipeline
### Scenario: A developer pushes code to GitHub. The app is tested, built into a Docker image, and deployed to a staging server.
```code
                 Developer pushes code to GitHub
                                 │
                                 ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  STAGE 1: TEST                                              │
    │                                                             │
    │  ┌─────────────────┐      ┌─────────────────────────────┐   │
    │  │  Job: Lint      │      │  Job: Unit & Integration    │   │
    │  │                 │      │  Tests                      │   │
    │  │  Step: checkout │      │  Step: checkout             │   │
    │  │  Step: npm lint │      │  Step: npm install          │   │
    │  └─────────────────┘      │  Step: npm test             │   │
    │      (parallel)           └─────────────────────────────┘   │
    │                               (parallel)                    │
    └───────────────────────┬─────────────────────────────────────┘
                            │ Both jobs must pass
                            ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  STAGE 2: BUILD                                             │
    │                                                             │
    │  Job: Docker Build & Push                                   │
    │                                                             │
    │  Step: checkout                                             │
    │  Step: docker build -t myapp:$GIT_SHA .                     │
    │  Step: docker push myrepo/myapp:$GIT_SHA                    │
    │  Step: store image tag as artifact                          │
    └───────────────────────┬─────────────────────────────────────┘
                            │ Image pushed to registry
                            ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  STAGE 3: DEPLOY TO STAGING                                 │
    │                                                             │
    │  Job: Deploy                                                │
    │                                                             │
    │  Step: pull image tag artifact                              │
    │  Step: SSH into staging server                              │
    │  Step: docker pull myrepo/myapp:$GIT_SHA                    │
    │  Step: docker compose up -d (with new image tag)            │
    │  Step: run smoke tests against staging URL                  │
    └─────────────────────────────────────────────────────────────┘
                            │
                            ▼
              ✅ Staging is live & tested
           (Manual approval gate → Production)
```

## Task 5: Explore in the Wild
**Repo:** [github.com/tiangolo/fastapi](https://github.com/tiangolo/fastapi)<br>
**Workflow file examined:** `.github/workflows/test.yml`
 
### What triggers it?
- `push` to the `master` branch
- Any `pull_request` opened against `master`
 
### How many jobs does it have?
**Ans.** Three jobs:
  1. `lint` — runs Ruff (Python linter) and mypy (type checker)
  2. `test` — runs pytest across a matrix of **Python versions** (3.8, 3.9, 3.10, 3.11, 3.12) and **OS combinations** (Ubuntu + macOS), creating ~10 parallel job instances
  3. `coverage` — aggregates test results and uploads a coverage report to Codecov
 
### What does it do?
**Ans.**
Every PR and merge to master triggers a full quality check: linting for code style, type checking for correctness, and a full test suite run across every supported Python version on multiple operating systems. This ensures FastAPI doesn't accidentally break compatibility with older Python versions and catches type errors that tests might miss. The coverage job ensures test coverage doesn't silently drop. A PR cannot be merged until all matrix jobs go green — which is exactly what CI is for.
 
---
 
## Key Takeaways
 
> **CI/CD is a practice, not a tool.** GitHub Actions, Jenkins, GitLab CI, and CircleCI are all tools that implement the same idea: automate the path from commit to deployable software.
 
> **A failing pipeline is not a problem — it's CI doing its job.** It caught something before it reached users.
 
> **The pipeline is the team's shared definition of "done."** If it passes, the code is tested, built, and ready. No individual's machine matters anymore.