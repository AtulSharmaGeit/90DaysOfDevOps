# GIT RESET VS REVERT & BRANCHING STRATEGIES

## Task 1: Git Reset
1. Create 3 commits (A, B, C)
    ```bash
    echo "Commit A" > file.txt
    git add file.txt
    git commit -m "Commit A"
    ```
    ```
    echo "Commit B" >> file.txt
    git add file.txt
    git commit -m "Commit B"
    ```
    ```
    echo "Commit C" >> file.txt
    git add file.txt
    git commit -m "Commit C"
    ```
    - We now have three commits in history: A → B → C.
    - `file.txt` contains lines from all three commits.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(15).png)

2. Go back one commit
    ```bash
    git reset --soft HEAD~1
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(16).png)

    - Moves HEAD back from C to B.
    - Commit C’s changes remain **staged** in the index.
    - If we run `git status`, we’ll see the changes ready to commit again.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(17).png)

    - Useful if we want to rewrite the commit message or combine commits.

3. Re-commit, then go back one commit (`reset --mixed`)
    ```bash
    git commit -m "Commit C"
    git reset --mixed HEAD~1
    ```
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(18).png)

    - HEAD moves back one commit again.
    - Commit C’s changes remain in the **working directory but unstaged**.
    - If we run `git status`, we’ll see modified files but not staged.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(19).png)

    - **We’d need `git add` again before committing.**
    - Useful if we want to edit files before recommitting.

4. Re-commit, then go back one commit (`reset --hard`)
    ```bash
    git add file.txt
    git commit -m "Commit C again"
    git reset --hard HEAD~1
    ```
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(20).png)

    - HEAD moves back one commit.
    - Commit C’s changes are **discarded completely** (both staged and unstaged).
    - Your working directory is reset to match commit B exactly.
    - This is destructive — changes are lost unless we recover via `git reflog`.
5. What is the difference between `--soft`, `--mixed`, and `--hard`?<br>
**Ans.**

     `git reset --soft HEAD~1`
    - Moves HEAD back one commit
    - Keeps all changes from the undone commit **staged** in the index.
    - Effect: We can immediately recommit with a new message or combine with other staged changes.

    `git reset --mixed HEAD~1` (default if no flag is given)
    - Moves HEAD back one commit.
    - Keeps changes from the undone commit in the **working directory but unstaged**.
    - Effect: We need to run `git add` again before committing.

    `git reset --hard HEAD~`1`
    - Moves HEAD back one commit.
    - Discards changes completely (both staged and unstaged).
    - Effect: Our working directory is reset to match the target commit exactly.
6. Which one is destructive and why?<br>
**Ans.** `--hard` - It erases changes from our working directory and index. Unless we recover them with git reflog, they are gone permanently.
7. When would we use each one?<br>
**Ans.** 

    `--soft`
    - When we want to rewrite a commit message or squash commits together.
    - Safe option because changes remain staged.

    `--mixed`
    - When we want to undo a commit but still keep the changes for editing.
    - Useful for reworking files before recommitting.

    `--hard`
    - When we want to completely discard changes and reset to a clean state.
    - Use only when we’re certain we don’t need the changes anymore.
8. Should we ever use `git reset` on commits that are already pushed?<br>
**Ans.** No.
    - Reset rewrites history, which causes conflicts for collaborators who already pulled the old commits.
    - For shared branches, use `git revert` instead — it preserves history and creates a new commit that undoes changes safely.

## Task 2: Git Revert
1. Make 3 commits (X, Y, Z)
    ```bash
    echo "Commit X" > file.txt
    git add file.txt
    git commit -m "Commit X"
    ```
    ```
    echo "Commit Y" >> file.txt
    git add file.txt
    git commit -m "Commit Y"
    ```
    ```
    echo "Commit Z" >> file.txt
    git add file.txt
    git commit -m "Commit Z"
    ```
    - We now have three commits in history: X → Y → Z.
2. Revert commit Y
    ```bash
    git log --oneline
    ```
    - Copy the commit hash of Y.
    ```bash
    git revert <commit-hash-of-Y>
    ```
    - Git creates a new commit that undoes the changes introduced by commit Y, without deleting Y itself.
3. Check the log
    ```bash
    git log --oneline
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/33b4b784cbcb5a5bc2391cf9e85a209513c4df3b/2026/day-25/Screenshots/Screenshot%20(21).png)

    - We’ll still see commit Y in the history.
    - A new commit appears at the end, usually labeled “Revert <commit Y>”.
4. How is git revert different from git reset?<br>
**Ans.** `git reset` moves HEAD and can remove commits from history, while `git revert` creates a new commit that undoes changes but keeps history intact.

5. Why is revert considered safer than reset for shared branches?<br>
**Ans.** Revert is safer for shared branches because it preserves history and avoids conflicts.

6. When would we use revert vs reset?<br>
**Ans.** Use reset for local cleanup before pushing, and revert when undoing changes in a branch that others may already be using.
## Task 3: Reset vs Revert

|                                  |                  `git reset`                 |                 `git revert`                  |
|----------------------------------|----------------------------------------------|-----------------------------------------------|
| What it does                     | Moves HEAD back, optionally discards changes | Creates a new commit that undoes changes      |
| Removes commit from history?     | Yes, rewrites history                        | No, history is preserved                      |
| Safe for shared/pushed branches? | ❌ No                                        | ✅ Yes                                       |
| When to use                      | Local cleanup, rewriting commits before push | Undo changes safely in shared/pushed branches |



## Task 4: Branching Strategies
1. **GitFlow**
- **How it works:**
    - Long-lived main and develop branches.
    - Features branch off develop.
    - Release branches created from develop.
    - Hotfix branches created from main.
- **Diagram:**
    ```Code
    main ← hotfix
    ↑
    release ← develop ← feature
    ```
- **When/where used:** Large teams, enterprise projects, scheduled releases.
- **Pros:** Clear structure, supports multiple versions, good for planned releases.
- **Cons:** Heavy process, slower for continuous delivery.

2. **GitHub Flow**
- **How it works:**
    - Single main branch.
    - Developers create short-lived feature branches.
    - Pull requests merged directly into main.
- **Diagram:**
    ```Code
    main ← feature
    ```
- **When/where used:** Startups, web apps, continuous deployment.
- **Pros:** Simple, fast, encourages frequent integration.
- **Cons:** Less control for complex release cycles, requires strong testing.

3. **Trunk-Based Development**
- **How it works:**
    - Everyone commits directly to main (the trunk).
    - Short-lived branches (hours/days) merged quickly.
    - Heavy reliance on automated testing and CI/CD.
- **Diagram:**
    ```Code
    main ← tiny branches
    ```
- **When/where used:** High-performing teams, CI/CD pipelines, rapid delivery environments.
- **Pros:** Very fast, fewer merge conflicts, promotes continuous integration.
- **Cons:** Requires discipline, strong automation, riskier without tests.

4. Which strategy would we use for a startup shipping fast?<br>
**Ans.** GitHub Flow - Simple, lightweight, and optimized for continuous deployment. Developers create short-lived feature branches, open pull requests, and merge directly into main.

    Alternative - Trunk-Based Development if the team has strong automated testing and CI/CD pipelines, commits go directly to main with very short-lived branches.

    Benefit: Enables rapid iteration and deployment without heavy branching overhead.
5. Which strategy would you use for a large team with scheduled releases?<br>
**Ans.** GitFlow - Provides structure with develop, release, feature, and hotfix branches. This allows parallel work on features while maintaining stable release cycles.

    Benefit: Clear separation of work streams, supports multiple versions, and reduces chaos in large-scale projects
6. Which one does your favorite open-source project use? (check any repo on GitHub)<br>
**Ans.** GitHub Flow - Work happens in short-lived feature branches, merged into main via pull requests. This keeps the project simple and encourages frequent contributions from the community.


