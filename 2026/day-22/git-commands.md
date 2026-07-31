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
![image alt]()

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
## Task 3: Git Commands Reference.

1. Setup & Config
- `git --version` → Check Git installation  
- `git config --global user.name "Atul"` → Set username  
- `git config --global user.email "atul@example.com"` → Set email  
- `git config --list` → View all config  


2. Basic Workflow
- `git init` → Initialize repo  
- `git add <file>` → Stage file  
- `git commit -m "message"` → Commit staged changes  

3. Viewing Changes
- `git status` → Show repo status  
- `git log` → Show commit history  
- `git log --oneline` → Compact history  

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

