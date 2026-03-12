# SHELL SCRIPTING CHEAT SHEET.

## QUICK REFERENCE TABLE


|  Topic   |        Key Syntax        |             Example                |
|----------|--------------------------|------------------------------------|
| Variable | `VAR="value"`            | `NAME="DevOps"`                    |
| Argument | `$1`, `$2`               | `./script.sh arg1`                 |
|    If    | `if [ condition ]; then` | `if [ -f file ]; then`             |
| For loop | `for i in list; do`      | `for i in 1 2 3; do`               |
| Function | `name() { ... }`         | `greet() { echo "Hi"; }`           |
|   Grep   | `grep pattern file`      | `grep -i "error" log.txt`          |
|   Awk    | `awk '{print $1}' file`  | `awk -F: '{print $1}' /etc/passwd` |
|   Sed    | `sed 's/old/new/g' file` | `sed -i 's/foo/bar/g' config.txt`  |

## 1. BASICS
1. Shebang (`#!/bin/bash`)
- **What it does:** Tells the system which interpreter to use when running the script.
- **Why it matters:** Ensures consistent execution regardless of user’s default shell.
```bash
#!/bin/bash
echo "This script runs with bash"
```

2. Running a Script
- `chmod +x script.sh`  - Make it executable.
- `./script.sh`         - Run it directly.
- `bash script.sh`      - Run with interpreter explicitly.<br>
  - Use `./script.sh` when your script has a proper shebang (`#!/bin/bash`).
  - Use `bash script.sh` if you want to force Bash to run it, even if the shebang is missing or points somewhere else.

3. Comments
- Single line.
```bash
# This is a comment
```
- Inline.
```bash
echo "Hello" # This prints Hello
```

4. Variables
- Declare and use:
```bash
VAR="Hello"
echo $VAR
```
- Quoting differences:
```bash
echo "$VAR"   # expands to Hello
echo '$VAR'   # literal $VAR
```
- Think of double quotes as saying something to a friend and letting them fill in the blanks:
  - You say: “My name is $VAR”
  - Your friend knows `$VAR = Atul`, so they reply: “My name is Atul.”
  - Use double quotes when you want variables to expand safely (especially when values may contain spaces).
- Now imagine single quotes as writing on a sticky note that says:
  - ‘My name is $VAR’
  - Your friend reads it exactly as written, without substituting anything. They’ll say: “My name is $VAR.”
  - Use single quotes when you want the text to stay literal, untouched.

5. Reading User Input
```bash
echo "Enter your name:"
read name
echo "Hi $name"
```
- `read name` - waits for the user to type something and stores it in the variable name.
- `echo "Hi $name"` - prints the stored value back.

We can also customize the prompt inline:
```bash
read -p "Enter your age: " age
echo "You are $age years old"
```
- `read` - waits for user input.
- `-p "Enter your age: "` - shows a prompt message directly in the terminal.
- `age` - the variable where the user’s input will be stored.
- `echo` - prints text to the terminal.
- `$age` - expands to the value you typed

6. Command-Line Arguments
```bash
echo "Script name: $0"    # ./script.sh
echo "First arg: $1"      # arg1
echo "Second arg: $2"     # arg2
echo "Arg count: $#"      # number of arguments (2 here)
echo "All args: $@"       # all arguments (arg1 arg2)
echo "Last exit code: $?" # status of last command
```
- When we run a script like:
```bash
./script.sh arg1 arg2
```
the shell automatically makes these arguments available inside the script.<br>
Suppose we have a script called `greet.sh`
```bash
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
```
Now run it:
```bash
./greet.sh Atul devops
```
Output:

    Script name: ./greet.sh
    First argument: Atul
    Second argument: devops

## 2. OPERATORS & CONDITIONALS
1. String Comparisons
- Used inside `[ ]` or `[[ ]]` to compare text.

```bash
a="hello"; b="world"

[ "$a"   =  "hello" ]   # true if equal
[ "$a"  !=  "$b"    ]   # true if not equal
[       -z  "$a"    ]   # true if string is empty
[       -n  "$a"    ]   # true if string is non-empty
```
Step-by-step Explanation
- `=` - Checks if two strings are equal.
  - `$a` is `"hello"`, so `[ "$a" = "hello" ]` is true.
- `!=` - Checks if two strings are not equal.
  - `$a` is `"hello"`, `$b` is `"world"`, so `[ "$a" != "$b" ]` is true.
- `-z` - True if the string length is zero (empty string).
  - `$a` is `"hello"`, so `[ -z "$a" ]` is false.
- `-n` - True if the string length is non-zero.
  - `$a` is `"hello"`, so `[ -n "$a" ]` is true.

2. Integer Comparisons
- Used for numeric values.
```bash
x=5; y=10

[ $x -eq $y ]   # true if equal (5 == 10 → false)
[ $x -ne $y ]   # true if not equal (5 != 10 → true)
[ $x -lt $y ]   # true if less than (5 < 10 → true)
[ $x -gt $y ]   # true if greater than (5 > 10 → false)
[ $x -le $y ]   # true if less or equal (5 <= 10 → true)
[ $x -ge $y ]   # true if greater or equal (5 >= 10 → false)
```

3. File Test Operators
- These operators let us check properties of files and directories inside conditionals (`[ ]` or `[[ ]]`).
```bash
[ -f file.txt ]   # true if file.txt is a regular file
[ -d /path ]      # true if /path is a directory
[ -e file.txt ]   # true if file.txt exists
[ -r file.txt ]   # true if file.txt is readable
[ -w file.txt ]   # true if file.txt is writable
[ -x file.txt ]   # true if file.txt is executable
[ -s file.txt ]   # true if file.txt is non-empty

```

4. If / Elif / Else Syntax
- This structure lets you control the flow of your script based on conditions.
```bash
if [ -f file.txt ]; then
  echo "File exists"
elif [ -d /path ]; then
  echo "Directory exists"
else
  echo "Nothing found"
fi
```
Step-by-step
- `if` - First condition is checked.
  - `[ -f file.txt ]` - true if `file.txt` is a regular file.
  - If true, it runs the block after `then`.
- `elif` - “Else if” - checked only if the first condition was false.
  - `[ -d /path ]` - true if `/path` is a directory.
  - If true, it runs this block.
- `else` - Runs if none of the above conditions were true.
  - Prints `"Nothing found"`.
- `fi` - Marks the end of the conditional block (like closing a bracket).

5. Logical Operators
- Logical operators let you combine conditions or invert them.
```bash

[ -f file.txt ] && echo "File exists"   # AND
[ -f file.txt ] || echo "File missing"  # OR
! [ -f file.txt ]                       # NOT
```
Step-by-step
- AND (`&&`)
  - Runs the second command only if the first succeeds.
  - Example: If `file.txt` exists, print `"File exists"`.
- OR (`||`)
  - Runs the second command only if the first fails.
  - Example: If `file.txt` does not exist, print `"File missing"`.
- NOT (`!`)
  - Inverts the condition.
  - Example: `! [ -f file.txt ]` - true if `file.txt` does not exist.

6. Case Statements
- Case statements are used for pattern matching when we have multiple possible options. They’re cleaner than writing a long chain of `if/elif/else`.
```bash
case $1 in
  start) echo "Starting service";;
  stop)  echo "Stopping service";;
  restart) echo "Restarting";;
  *)     echo "Unknown option";;
esac
```
Step-by-step
- `case $1 in` - Look at the first argument passed to the script.
- `start)` - If `$1` equals `"start"`, run this block.
- `stop)` - If `$1` equals `"stop"`, run this block.
- `restart)` - If `$1` equals `"restart"`, run this block.
- `*)` - Default case (like “else”), runs if none of the above match.
- `esac` - Marks the end of the case statement (like `fi` for `if`).

## 3. LOOPS
1. `for` Loop -- List-Based
- A `for` loop lets you iterate over a list of values one by one.
```bash
for item in apple banana cherry; do
  echo $item
done
```
Step-by-step
- `for item in apple banana cherry; do`
  - The loop starts, and `item` takes the value `"apple"` first.
- `echo $item`
  - Prints `"apple"`.
- Next iteration - `item="banana"`, prints `"banana"`.
- Next iteration - `item="cherry"`, prints `"cherry"`.
- `done` - Marks the end of the loop.

2. `for` Loop — C-Style
- This loop uses a syntax similar to the **C programming language**. It’s useful when we want to iterate over a numeric range with explicit control over the counter.
```bash
for ((i=0; i<5; i++)); do
  echo $i
done
```
Step-by-step
- Initialization (`i=0`) → Start with `i = 0`.
- Condition (`i<5`) → Keep looping while `i` is less than 5.
- Increment (`i++`) → After each loop, increase `i` by 1.
- Body (`echo $i`) → Runs each time with the current value of `i`.
- `done` → Marks the end of the loop.

Output:

    0
    1
    2
    3
    4

3. `while` Loop
- A `while` loop runs **as long as the condition is true**. Once the condition becomes false, the loop stops.
```bash
x=1
while [ $x -le 5 ]; do
  echo $x
  ((x++))
done
```
Step-by-step
- Initialization (`x=1`) → Start with `x = 1`.
- Condition (`[ $x -le 5 ]`) → Loop continues while `x` is less than or equal to 5.
- Body (`echo $x`) → Prints the current value of `x`.
- Increment (`((x++))`) → Increase `x` by 1 each time.
- When `x` becomes 6, the condition fails (`6 -le 5` is false), so the loop ends.

Output:

    1
    2
    3
    4
    5
4. `until` Loop
- An `until` loop runs until the condition becomes true.
It’s basically the opposite of a `while` loop.
```bash
x=1
until [ $x -gt 5 ]; do
  echo $x
  ((x++))
done
```
Step-by-step
- Initialization (`x=1`) → Start with `x = 1`.
- Condition (`[ $x -gt 5 ]`) → The loop continues **while this condition is false**.
  - At first, `1 -gt 5` is false, so the loop runs.
- Body (`echo $x`) → Prints the current value of `x`.
- Increment (`((x++))`) → Increase `x` by 1 each time.
- When `x` becomes 6, the condition `[ $x -gt 5 ]` is true, so the loop stops.

Output:

    1
    2
    3
    4
    5

5. Loop Control — `break` and `continue`
- We can control how loops behave using `break` and `continue`.
```bash
for i in 1 2 3 4 5; do
  if [ $i -eq 3 ]; then
    continue   # skip 3
  fi
  if [ $i -eq 4 ]; then
    break      # stop at 4
  fi
  echo $i
done
```
Step-by-step
- Loop starts with values `1 2 3 4 5`.
- When `i=1` → prints `1`.
- When `i=2` → prints `2`.
- When `i=3` → `continue` skips the rest of the loop body, so nothing is printed.
- When `i=4` → `break` stops the loop entirely, so the loop ends.
- `i=5` is never reached because the loop was broken at 4.

Output:

    1
    2
    5

6. Looping Over Files - `for file in *.log`
- We can use a `for` loop with a **wildcard pattern** to process multiple files at once.
```bash
for file in *.log; do
  echo "Processing $file"
done
```
Step-by-step
- `for file in *.log; do`
  - The shell expands `*.log` into all files ending with `.log` in the current directory.
  - Each filename is assigned to the variable `file` one by one.
- `echo "Processing $file"` - Prints the current filename.
- `done` - Marks the end of the loop.

7. Looping Over Command Output - `while read line`
- We can use a `while read` loop to process the **output of another command line by line**.
```bash
ls -1 | while read line; do
  echo "File: $line"
done
```
Step-by-step
- `ls -1` → Lists files, one per line.
- **Pipe (`|`)** → Sends that output into the next command (`while read line`).
- `while read line; do ... done` → Reads each line of input one at a time.
  - First iteration: `line="a.txt"` → prints `File: a.txt`.
  - Second iteration: `line="b.txt"` → prints `File: b.txt`.
- Loop ends when there are no more lines to read.

## 4. FUNCTIONS

1. Defining a Function
- Functions let us group reusable code blocks under a name, so we can call them multiple times without rewriting the same logic.
```bash
greet() {
  echo "Hello, DevOps!"
}
```
- `greet()` → Defines a function named `greet`.
- `{ ... }` → Contains the commands that run when the function is called.
- `echo "Hello, DevOps!"` → The body of the function.

2. Calling a Function
- Simply use the function name.
```bash
greet() {
  echo "Hello, DevOps!"
}

greet   # calls the function
```

3. Passing Arguments
- Just like scripts, functions can accept arguments. Inside the function, you access them using *positional parameters*:
  - `$1` - first argument
  - `$2` - second argument
  - `$3` - third argument, and so on
  - `$@` - all arguments
  - `$#` - number of arguments
```bash
say_hello() {
  echo "Hello $1, welcome to $2"
}
say_hello "Atul" "DevOps"
# Output: Hello Atul, welcome to DevOps
```
Step-by-step
- Define function - `say_hello() { ... }`.
- Call function with arguments - `say_hello "Atul" "DevOps"`.
- Inside the function:
  - `$1` = `"Atul"`
  - `$2` = `"DevOps"`
- The function prints: `"Hello Atul, welcome to DevOps"`.

4. Return Values — `return` vs `echo`
- `return`: Sets the exit status of the function (an integer between 0–255).
  - It does not output data directly.
  - Useful for signaling success/failure codes, not for returning actual values.
```bash
add() {
  return $(($1 + $2))
}
add 2 3
echo $?   # prints 5 (exit code)
```
  - `return 5` sets the function’s exit status to `5`.
  - `$?` captures the exit status of the last command (the function).
  - Prints `5`.
  - Limitation: Exit codes are restricted to **0–255**. If you try `return 300`, it wraps around modulo 256.

- `echo`: Prints data to **stdout** (standard output).
  - You can capture this output into a variable using command substitution (`$(...)`).
  - This is the usual way to “return” values in Bash.
```bash
sum() {
  echo $(($1 + $2))
}
result=$(sum 2 3)
echo $result   # prints 5
```
- The function prints `5`.
- `$(sum 2 3)` captures that output into the variable `result`.
- Prints `5`.

5. Local Variables
- By default, variables in Bash are **global** — meaning they can be accessed anywhere in the script once defined.
- Using `local` inside a function restricts the variable’s scope to that function only.

```bash
demo() {
  local msg="Inside function"
  echo $msg
}
demo
echo $msg   # empty, not accessible outside
```
Step-by-step
- `local msg="Inside function"` - Defines `msg` only inside `demo()`.
- `echo $msg` inside function - Prints `"Inside function"`.
- Outside function - `$msg` is not available, so nothing is printed.

Output:

    Inside function
(The second `echo $msg` prints nothing because `msg` was local to the function.)

## 5. TEXT PROCESSING COMMANDS
1. `grep` - It is used to **search for patterns in text**. It’s one of the most powerful text-processing tools in Linux.
- Flags:
    - `-i` - Ignore case (case-insensitive search).
    - `-r` - Recursive search through directories.
    - `-c` - Count the number of matches.
    - `-n` - Show line numbers with matches.
    - `-v` - Invert match (show lines that don’t match).
    - `-E` - Use extended regular expressions (regex).
```bash
grep "error" log.txt          # search for "error" in log.txt
grep -i "error" log.txt       # case-insensitive search, don’t care if it’s “Error” or “ERROR”.
grep -r "error" /var/log      # search recursively in /var/log, check every book in the library.
grep -c "error" log.txt       # count matches, just count how many times “error” appears.
grep -n "error" log.txt       # show line numbers, note the page number (line number).
grep -v "error" log.txt       # show lines without "error", look for all pages without “error”.
grep -E "err|warn" log.txt    # match "err" OR "warn", search for multiple words at once (“err” or “warn”).
```

2. `awk` - It is a powerful text-processing tool. It works by scanning input line by line, splitting each line into **fields (columns)**, and applying actions based on patterns.

- Common uses:

    1. *Print columns*
        ```bash
        awk '{print $1}' file.txt
        ```
        - Prints the **first column** of each line.
        - `$1` = first field, `$2` = second field, etc.
    2. *Change field separator*
        ```bash
        awk -F: '{print $1,$3}' /etc/passwd
        ```
        - `-F:` sets the field separator to `:`.
        - Prints the **first and third** fields from `/etc/passwd`.
    3. *Match patterns*
        ```bash
        awk '/error/ {print $0}' log.txt
        ```
        - Prints lines containing `"error"`.
        - `$0` = the entire line.
    4. *BEGIN/END blocks*
        ```bash
        awk 'BEGIN{print "Start"} END{print "End"}' file.txt
        ```
        - `BEGIN` runs before processing any lines.
        - `END` runs after all lines are processed.
        - Useful for headers, footers, or summaries.


3. `sed` —  It is a **stream editor** used to perform text transformations on input streams (like files or piped data). It’s especially handy for **search-and-replace, deleting lines, and in-place editing**.

- Common uses:

    1. Substitution
        ```bash
          sed 's/foo/bar/g' file.txt
          ``` 
        - Replaces all occurrences of `"foo"` with `"bar"` in `file.txt`.
        - `s` = substitute, `g` = global (replace all matches in the line).

    - Delete lines
      ```bash
      sed '/error/d' file.txt
      ```
      - Deletes all lines containing `"error"`.
      - `d` = delete.

    - In-place edit
      ```bash
      sed -i 's/foo/bar/g' config.txt
      ```
      - Replaces `"foo"` with `"bar"` directly inside `config.txt`.
      - `-i` = edit the file in place (no need to redirect output).

4. `cut` - It is used to **extract specific columns (fields) or character ranges** from text. It’s simple but very effective for parsing structured files like `/etc/passwd` or CSVs.


```bash
cut -d: -f1 /etc/passwd
cut -d, -f2 data.csv
```
- **Extract first field from** `/etc/passwd`.
  - `-d:` - delimiter is `:`
  - `-f1` - first field
  - Prints all usernames from `/etc/passwd`.
- **Extract second field from CSV**
  - `-d`, → delimiter is `,`
  - `-f2` → second field
  - Prints the second column (e.g., email addresses, scores, etc.).

5. `sort` - It arranges lines of text files in a specified order. By default, it sorts alphabetically.
- Flags:
    - `-n` - Numeric sort (treats values as numbers instead of text).
    - `-r` - Reverse order.
    - `-u` - Unique (removes duplicates).
```bash
sort names.txt      # Sorts names in ascending (A–Z) order.
sort -n numbers.txt # Sorts numbers correctly (e.g., 2, 10, 100 instead of 10, 100, 2).
sort -r names.txt   # Sorts names in descending (Z–A) order.
sort -u names.txt   # Sorts names and removes duplicates.
```

6. `uniq` - It filters out adjacent duplicate lines in a file or stream. It’s often used after `sort`, because duplicates must be consecutive for `uniq` to detect them.
- Flags
    - `-c` - Prefix each line with the number of occurrences.
```bash
uniq file.txt     # Prints each line once, removing consecutive duplicates.
uniq -c file.txt  # Shows how many times each line appeared.
```

7. `tr`(translate) - It is used to **transform or delete characters** from input streams. It works character-by-character, not word-by-word.
```bash
tr 'a-z' 'A-Z' < file.txt # Converts all lowercase letters (a-z) to uppercase (A-Z). Eg. "hello world" → "HELLO WORLD".
tr -d '0-9' < file.txt    # Deletes all digits (0–9) from the file. Eg. "abc123xyz" → "abcxyz".
```

8. `wc`(word count) - It is used to count lines, words, and characters in a file or input stream.

```bash
wc -l file.txt   # Count lines
wc -w file.txt   # Count words
wc -c file.txt   # Count characters (bytes)
```

9. `head` / `tail` — First/Last N Lines

- Flags.

    - `-n` - number of lines

    - `-f` - follow mode

```bash
head -n 10 file.txt # Shows the first 10 lines of file.txt. Default is 10 lines if -n is not specified.
tail -n 10 file.txt # Shows the last 10 lines of file.txt. Default is 10 lines if -n is not specified.
tail -f log.txt # Continuously displays new lines added to log.txt. Commonly used for monitoring live logs (e.g., web server logs).
```

## 6. USEFUL PATTERNS & ONE-LINERS
1. Find and Delete Files Older Than N Days
```bash
find /var/log -type f -mtime +7 -delete
```
Breakdown:
- `find /var/log` - Search inside `/var/log`.
- `-type f` - Only look for files (ignore directories).
- `-mtime +7` - Match files **modified more than 7 days ago**.
- `-delete` - Delete those files.

Variations
- Dry run (see files before deleting):
  ```bash
  find /var/log -type f -mtime +7
  ```
- Delete files older than 30 days:
  ```bash
  find /var/log -type f -mtime +30 -delete
  ```
- Delete empty files only:
  ```bash
  find /var/log -type f -empty -delete
  ```

2. Count Lines in All `.log` Files
```bash
wc -l *.log
```
- `*.log` → expands to all files ending with `.log` in the current directory.
- `wc -l` → counts the number of lines in each file.
- Output shows line counts per file, plus a total at the end.

Example Output

    120 app.log
    85 error.log
    50 access.log
    255 total

Variations
- Count words in all `.log` files:
  ```bash
  wc -w *.log
  ```
- Count characters in all `.log` files:
  ```bash
  wc -c *.log
  ```
- Count lines across all `.log` files together (no per-file breakdown):
  ```bash
  cat *.log | wc -l
  ```

3. Replace a String Across Multiple Files
```bash
sed -i 's/oldstring/newstring/g' *.txt
```
Breakdown
- `sed` - stream editor.
- `-i` - in-place edit (changes files directly).
- `s/oldstring/newstring/g` - substitute `oldstring` with `newstring` globally (all occurrences in each line).
- `*.txt` - applies to all `.txt` files in the current directory.

Variations
- Preview changes (no in-place edit):
  ```bash
  sed 's/oldstring/newstring/g' *.txt
  ```
  - Prints modified content to stdout without changing files.
- Backup before editing:
  ```bash
  sed -i.bak 's/oldstring/newstring/g' *.txt
  ```
  - Creates backup files with .bak extension before editing.
- Recursive replacement in subdirectories:
  ```bash
  find . -name "*.txt" -exec sed -i 's/oldstring/newstring/g' {} +
  ```

4. Check if a Service is Running
```bash
ps aux | grep nginx
```
Breakdown
- `ps aux` → Lists all running processes with details (user, PID, CPU/mem usage, command).
- `| grep nginx` → Filters the list to show only processes containing `"nginx"`.
- If you see output, it means **nginx is running**.
- If nothing appears, nginx is not running.

Variations
- Avoid matching the grep process itself:
  ```bash
  ps aux | grep [n]ginx
  ```
  - This trick prevents the grep nginx line from appearing in results.
- Check by process name directly:
  ```bash
  pgrep nginx
  ```
  - Returns the process IDs (PIDs) of running nginx processes.
- Check with systemd (modern Linux):
  ```bash
  systemctl status nginx
  ```
  - Shows detailed service status (running, stopped, failed).

5. Monitor Disk Usage with Alerts
```bash
df -h | awk '$5+0 > 80 {print "ALERT: "$0}'
```
Breakdown
- `df -h` - Shows disk usage in human-readable format (GB/MB).
- `$5` - Refers to the 5th column (percentage used).
- `+0` - Converts the percentage string (like `85%`) into a number for comparison.
- `> 80` - Condition: if usage is greater than 80%.
- `print "ALERT: "$0` - Prints the entire line with an “ALERT:” prefix.

Variations
- Alert at 90% usage:
  ```bash
  df -h | awk '$5+0 > 90 {print "CRITICAL: "$0}'
  ```
- Send email alert (example with mail):
  ```bash
  df -h | awk '$5+0 > 80 {print "Disk Alert: "$0}' | mail -s "Disk Usage Alert" admin@example.com
  ```
- Run as a cron job (every hour):
  ```bash
  0 * * * * df -h | awk '$5+0 > 80 {print "ALERT: "$0}' >> /var/log/disk_alerts.log
  ```

6. Parse CSV (First Column)
```bash
cut -d, -f1 data.csv
```
Breakdown
- `cut` → Extracts sections of each line.
- `-d,` → Sets the delimiter to a comma (`,`), since CSV files are comma-separated.
- `-f1` → Selects the **first field (column)**.
- `data.csv` → Input file.

Variations
- Extract second column (emails):
  ```bash
  cut -d, -f2 data.csv
  ```
- Extract multiple columns (name + score):
  ```bash
  cut -d, -f1,3 data.csv
  ```
- Extract character ranges (not fields):
  ```bash
  cut -c1-5 data.csv
  ```
  - Shows only the first 5 characters of each line.

7. Tail a Log and Filter for Errors in Real Time
```bash
tail -f app.log | grep "ERROR"
```
Breakdown
- tail -f app.log → Streams new lines as they are appended to app.log.

- | grep "ERROR" → Filters only lines containing "ERROR".

- Together, this means you’ll see live error entries as they happen.

Variations
- Case-insensitive search:
  ```bash
  tail -f app.log | grep -i "error"
  ```
- Highlight matches (with grep --color):
  ```bash
  tail -f app.log | grep --color=auto "ERROR"
  ```
- Multiple keywords (errors or warnings):
  ```bash
  tail -f app.log | grep -E "ERROR|WARN"
  ```
- Count errors in real time (less common but possible):
  ```bash
  tail -f app.log | grep --line-buffered "ERROR" | awk '{count++} {print count, $0}'
  ```

## 7. ERROR HANDLING & DEBUGGING
1. Exit Codes - Every command in Linux/Unix returns an exit status (an integer).
  - `$?`      - exit status of last command (0 = success, non-zero = error).
  - `exit 0`  - script exits successfully.
  - `exit 1`  -  script exits with error, different numbers can indicate different error types.
- Successful command
  ```bash
  ls /tmp
  echo $?
  ```
  Output:

      0
  - Means the command succeeded.
- Failed command
  ```bash
  ls /notfound
  echo $?
  ```
  Output:

      2
  - (or another non-zero code depending on the error).
  - This indicates failure.
- Explicit exit codes in scripts
  ```bash
  exit 0   # script exits successfully
  exit 1   # script exits with error
  ```

2. `set -e` - Exit on Error.
- When enabled, the script will **immediately exit** if any command returns a **non-zero exit code (failure)**.
- This prevents the script from continuing after an error, which could otherwise cause unintended consequences.
```bash
set -e
cp file.txt /backup/   # if this fails, script exits
echo "This won't run if cp fails"
```
- If `cp` succeeds - the script continues.
- If `cp` fails (e.g., file not found, permission denied) - the script stops right there, and the `echo` line never runs.

3. `set -u` - Treat Unset Variables as Error
- When enabled, Bash will **exit immediately** if you try to use an **undefined (unset) variable**.
- This prevents subtle bugs where a typo or missing variable could cause unexpected behavior.
```bash
set -u
echo $UNDEFINED_VAR   # script exits with error
```
- Since `UNDEFINED_VAR` is not defined, the script stops with an error.
- Without `set -u`, Bash would just print an empty string and continue, which could hide problems.

4. `set -o pipefail` - Catch Errors in Pipes
- By default, in a pipeline (`cmd1 | cmd2 | cmd3`), Bash only returns the exit code of the **last command**.
- With `set -o pipefail`, the pipeline’s exit code will be the **first non-zero exit code** in the chain.
- This ensures that if **any command fails**, the whole pipeline is considered failed.
```bash
set -o pipefail
cat file.txt | grep "error" | sort
# If grep fails, whole pipeline fails
```
- If `grep` fails (e.g., no match, or file missing), the pipeline fails.
- Without `pipefail`, the pipeline might still exit with `0` if `sort` succeeds, hiding the error.

Best Practices
- Combine with strict mode:
  ```bash
  set -euo pipefail
  ```
  - `-e` → exit on error
  - `-u` → undefined variable error
  - `-o pipefail` → fail if any command in a pipeline fails
  - Useful in **deployment scripts, CI/CD pipelines, and backups**, where silent failures can cause big issues.

5. `set -x` - Debug Mode
- Enables **debugging/tracing mode**.
- Bash prints each command (and its arguments) to **stderr** before executing it.
- Useful for understanding script flow and troubleshooting errors.
```bash
set -x
echo "Debugging script"
set +x   # turn off debug mode
```
Output (example):

    + echo 'Debugging script'
    Debugging script
- The `+` prefix shows the command being executed.

- After `set +x`, tracing stops, so subsequent commands run silently again.

Best Practices
- Use `set -x` at the start of a script for full tracing:
```bash
#!/bin/bash
set -euxo pipefail
```
- Or wrap only critical sections:
```bash
set -x   # start tracing
cp file.txt /backup/
set +x   # stop tracing
```
- Combine with logging (`exec 2>debug.log`) to capture debug output:
```bash
set -x
exec 2>debug.log
```

6. Trap - Cleanup on Exit
- It lets us run custom commands automatically when a script receives certain signals or exits.
- Common use case: cleanup tasks (removing temp files, stopping services, logging).
```bash
trap 'echo "Cleaning up..."; rm -f temp.txt' EXIT

echo "Doing work..."
# When script ends, cleanup runs automatically
```
- `EXIT` - special signal that triggers when the script finishes.
- The command inside quotes runs just before the script terminates.
- In this case, it prints a message and deletes `temp.txt`.

Other Useful Signals
- INT → Triggered when you press Ctrl+C.
- TERM → Triggered when the process is terminated.
- EXIT → Triggered when the script ends (success or failure).
Example:
```bash
trap 'echo "Interrupted!"; cleanup_function' INT
```
Best Practices
- Use trap for:
  - Removing temp files.
  - Stopping background jobs.
  - Logging script exit status.
  - Rolling back partial changes in deployments.
Combine with strict mode:
```bash
set -euo pipefail
trap 'echo "Script failed at line $LINENO"; cleanup' ERR EXIT
```
# THE END!