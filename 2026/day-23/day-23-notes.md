# GIT BRANCHING & WORKING WITH GITHUB

## Task 1: Understanding Branches.
1. What is a branch in Git?  
    **Ans.** A branch is a movable pointer to a commit. It represents an independent line of development, allowing you to work on features, fixes, or experiments without affecting the main branch.
2. Why do we use branches instead of committing everything to main?  
    **Ans.** Branches isolate changes. This prevents breaking production code, enables parallel development, and makes collaboration easier by keeping feature work separate until it’s ready to merge.
3. What is `HEAD` in Git?  
    **Ans.** `HEAD` is a pointer that represents your current checked-out commit. Usually, it points to the latest commit on the branch you’re working on.
4. What happens to your files when you switch branches?  
    **Ans.** Git updates your working directory to match the snapshot of the commit that branch points to. Files may change, disappear, or reappear depending on differences between branches.

## Task 2: Branching Commands (to add in git-commands.md)
1. List all branches
    ```
    git branch
    ```
2. Create a new branch
    ```
    git branch feature-1
    ```
3. Switch to a branch
    ```bash
    git checkout feature-1
    ```
4. Create and switch in one command
    ```bash
    git checkout -b feature-2
    ```
5. Modern alternative to checkout
    ```bash
    git switch feature-1
    git switch -c feature-3   # create + switch
    ```
    - `git switch` is safer and clearer than `git checkout` because it’s dedicated to branch switching, while checkout has multiple roles (branch switching + file restore).
6. Make a commit on feature-1
    ```bash
    echo "Feature 1 work" >> feature1.txt
    git add feature1.txt
    git commit -m "Add feature1 work"
    ```
7. Switch back to main
    ```bash
    git switch main
    ```
8. Delete a branch
    ```bash
    git branch -d feature-2
    git branch -d feature-3
    ```

## Task 3: Push to GitHub
1. Add remote repo to GitHub.
```bash
git remote add origin https://github.com/AtulSharmaGeit/devops-git-practice.git
```
2. Push main branch
```bash
git push -u origin main
```
3. Push feature-1 branch
```bash
git push -u origin feature-1
```
What is the difference between `origin` and `upstream`?
- origin - your fork or personal copy of the repo (where you push your work).
- upstream - the original repository you forked from (used to sync your fork with the source).

## Task 4: Pull from GitHub
```bash
git pull origin main
```
What is the difference between `git fetch` and `git pull`?
- git fetch downloads changes but doesn’t merge them into your working branch.
- git pull = git fetch + git merge (it updates your branch immediately).

## Task 5: Clone vs Fork
1. Clone a public repo
    ```bash
    git clone https://github.com/someuser/somerepo.git
    ```
2. Fork the repo on GitHub, then clone your fork
    ```bash
    git clone https://github.com/<your-username>/somerepo.git
    ```
3. What is the difference between clone and fork?
    - Clone - makes a local copy of a repo.
    - Fork - makes a copy of someone else’s repo under your GitHub account.
4. When would you clone vs fork?
    - Clone → when you just want to work locally or contribute directly.
    - Fork → when you want your own independent copy to experiment or submit pull requests.

5. After forking, how do you keep your fork in sync with the original repo?
    - Add the original repo as upstream and pull changes regularly.

    ```bash
    git remote add upstream https://github.com/someuser/somerepo.git
    git fetch upstream
    git merge upstream/main
    ```