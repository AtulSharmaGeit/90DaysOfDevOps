# INTRODUCTION TO GIT

## Task 1: Install and Configure Git

1. Verify installation
    ```bash
    git --version
    ```
2. Set identity
    ```bash
    git config --global user.name "Atul"
    git config --global user.email "atul@example.com"
    ```
3. Verify configuration
    ```bash
    git config --list
    ```

## Task 2: Create Your Git Project
1. Create folder:
    ```bash
    mkdir devops-git-practice
    cd devops-git-practice
    ```
2. Initialize repo:
    ```bash
    git init
    ```
3. Check status:
    ```bash
    git status
    ```
4. Explore `.git`:
    ```bash
    ls -a or ls -Force
    cd .git
    ls
    ```
![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ce99e4726f3cdc5062284f52aacfb2e355502b43/2026/day-22/Screenshots/Screenshot%20(538).png)

- `hooks/`<br>
Contains sample scripts that Git can run automatically on certain actions (e.g., before a commit, after a push). You can customize these to enforce rules or automate tasks.
- `info/`  
Holds miscellaneous info, like the exclude file where you can define ignore patterns (similar to .gitignore, but local only).
- `objects/`  
This is the actual database of Git. Every commit, file snapshot, and tree is stored here as compressed objects. It’s the core of Git’s version control.
- `refs/`  
Stores references to commits — like branches (refs/heads/) and tags (refs/tags/). When you check out main, Git looks here to see which commit it points to.
- `config`  
Repo-specific configuration file (different from your global config). For example, you can set a different username/email just for this repo.
- `description`  
Used by GitWeb (a web interface for Git repos). Not important for local work.
- `HEAD`  
A pointer to your current branch. Right now it probably says ref: refs/heads/main. This tells Git “you’re on the main branch.”

## Task 3: Creating Git Commands Reference.

- Create and add `git_commands.md`
   ```bash
   echo "# Git Commands Reference" > git-commands.md
   vim git-commands.md
   ```
```Markdown
# Git Commands Reference

## Setup & Config
- `git --version`  
  *Checks if Git is installed and shows the version.*  

- `git config --global user.name "AtulSharmaGeit"`  
  *Sets your global username for commits.*  

- `git config --global user.email "s.atul47@yahoo.in"`  
  *Sets your global email for commits.*  

- `git config --list`  
  *Displays all current Git configuration settings.*  

---

## Basic Workflow
- `git init`  
  *Initializes a new Git repository in the current folder.*  

- `git add git-commands.md`  
  *Stages changes to be committed.*  

- `git commit -m "Initial commit"`  
  *Commits staged changes with a descriptive message.*  

---

## Viewing Changes
- `git status`  
  *Shows the current state of the working directory and staging area.*  

- `git log`  
  *Displays the commit history with details.*  

- `git log --oneline`  
  *Shows a compact version of the commit history.*
```
- Stage and commit:
   ```bash
   git add git-commands.md
   git commit -m "Add initial Git commands reference"
   ```
- Verify history:
   ```bash
   git log  
   ```
## Task 4: Stage and Commit
1. Stage file:
```bash
git add git-commands.md
```
2. Check staged:
```bash
git status
```
3. Commit:
```bash
git commit -m "Add initial Git commands reference"
```
4. View history:
```bash
git log
```
## Task 5: Understand the Git Workflow
1. **What is the difference between `git add` and `git commit`?**  
   - `git add` moves changes into the staging area.  
   - `git commit` saves those staged changes into the repository history.

2. **What does the staging area do? Why not commit directly?**  
   - The staging area lets us prepare and review changes before committing.  
   - It gives control to commit only selected changes instead of everything at once.

3. **What information does `git log` show you?**  
   - Commit history: commit IDs, author, date, and commit messages.

4. **What is the `.git/` folder and what happens if you delete it?**  
   - `.git/` stores all repo metadata, history, and configuration.  
   - If deleted, the folder stops being a Git repo — you lose version history.

5. **Difference between working directory, staging area, and repository**  
   - **Working directory**: Your actual files on disk.  
   - **Staging area**: Snapshot of changes you plan to commit.  
   - **Repository**: Permanent history of committed changes.
