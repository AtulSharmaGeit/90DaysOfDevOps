# YAML Basics

## Task 1: Key-Value Pairs
1. Create the file
    ```bash
    vim person.yaml
    ```
2.  Write key-value pairs
    ```yaml
    name: Atul
    role: DevOps Engineer
    experience_years: 2
    learning: true
    ```
    - Use spaces, never tabs.
    - Booleans are written as `true` or `false` (without quotes).
    - Strings don’t need quotes unless they contain special characters like `:` or `#`.
3. Verify the file
    ```bash
    cat person.yaml
    ```
    We should see:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(277).png)
    
    Check:
    - No tabs (only spaces).
    - Clean alignment.
    - No trailing spaces.
4. Validate

    Install yamllint (if not already):
    ```bash
    sudo apt install yamllint
    ```
    Run:
    ```bash
    yamllint person.yaml
    ```
    If everything is correct, we’ll see no errors.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(279).png)

## Task 2: Lists
1. Open our file
    ```bash
    vim person.yaml
    ```
2. Add a block-style list for tools
    ```yaml
    tools:
      - docker
      - kubernetes
      - terraform
      - ansible
      - jenkins
    ```
    - Indentation matters: 2 spaces before each `-`.
    - No tabs - only spaces.
3. Add an inline-style list for hobbies
    ```yaml
    hobbies: [reading, cycling, coding]
    ```
4. Verify
    ```bash
    cat person.yaml
    ```
    We should see something like:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(280).png)

5. Validate
    ```bash
    yamllint person.yaml
    ```
    If indentation is correct, we’ll see no errors.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(279).png)

6. What are the two ways to write a list in YAML?

    **Ans.**

    1. Block style (each item on its own line with `-`):
        ```yaml
        tools:
          - docker
          - kubernetes
        ```
    2. Inline style (items inside square brackets, comma-separated):
        ```yaml
        hobbies: [reading, cycling, coding]
        ```

## Task 3: Nested Objects
1. Create the file
```bash
vim server.yaml
```
2. Add nested objects
    ```yaml
    server:
      name: app-server
      ip: 192.168.1.10
      port: 8080

    database:
      host: db.local
      name: appdb
      credentials:
        user: admin
        password: secret123
    ```
    - Each nested level is indented with 2 spaces.
    - `server` and `database` are top-level keys.
    - `credentials` is nested inside `database`.
3. Verify
    ```bash
    cat server.yaml
    ```
    We should see the clean YAML structure.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(283).png)

4. Validate
    ```bash
    yamllint server.yaml
    ```
    If indentation is correct, we’ll see no errors.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(284).png)

5. What happens if we use a tab?

    **Ans.** Try editing the file and replace one indentation with a tab instead of spaces.
    
    For example:
    ```yaml
    credentials:
        user: admin
        password: secret123   # ← tab used here
    ```
    Now validate:
    ```bash
    yamllint server.yaml
    ```
    We’ll get an error like:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(285).png)

    YAML does not allow tabs - only spaces. That’s why indentation errors are the most common issue when writing YAML.

## Task 4: Multi-line Strings
1. Open `server.yaml`
    ```bash
    vim server.yaml
    ```
2. Add a startup script using block style (`|`)

    This preserves newlines exactly as written:
    ```yaml
    startup_script: |
      echo "Starting server"
      systemctl start nginx
      echo "Server started successfully"
    ```
    - If we print this value later, it will keep the line breaks exactly as shown.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(286).png)

3. Add a startup script using fold style (`>`)

    This folds lines into a single string, replacing newlines with spaces:
    ```yaml
    startup_script_folded: >
    echo "Starting server"
    systemctl start nginx
    echo "Server started successfully"
    ```
    - If we print this value later, it will appear as:

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(287).png)

4. Verify
    ```bash
    cat server.yaml
    ```
    We should see both versions of the script.

5. Validate
    ```bash
    yamllint server.yaml
    ```
    If indentation is correct, no errors.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/ac142f43608dcc555a8a8d0e05a34a0b93f498af/2026/day-38/Screenshots/Screenshot%20(288).png)

6. When would we use `|` vs `>`?

    **Ans.**
    - Use `|` **(block style)** when we want to preserve exact formatting — e.g., shell scripts, configuration files, or text where line breaks matter.
    - Use `>` **(fold style)** when we want long text folded into a single line — e.g., documentation, descriptions, or log messages.

## Task 5: Spot the Difference
### Block 1 (Correct)
```yaml
name: devops
tools:
  - docker
  - kubernetes
```
- `tools` is a key.
- Each list item (`docke`r, `kubernetes`) is indented 2 spaces under `tools`.
- Clean, consistent indentation.

### Block 2 (Broken)
```yaml
name: devops
tools:
- docker
  - kubernetes
```
What’s wrong:
- Indentation mismatch
    - The first item (`docker`) is aligned directly under `tools` with no indentation.
    - The second item (`kubernetes`) is indented with 2 spaces, making YAML think it belongs to a different nested list.
- Structural confusion
    - YAML interprets this as:
        ```yaml
        tools:
          - docker
            - kubernetes
        ```
        Which is invalid, because `docker` is a string, not a parent object that can contain another list.

- In YAML, all list items under the same key must be indented consistently.
    - Correct way:
        ```yaml
        tools:
          - docker
          - kubernetes
        ```