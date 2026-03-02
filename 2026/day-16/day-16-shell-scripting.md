# Shell Scripting Basics

## Task 1: First Script

`vim hello.sh`

```bash
#!/bin/bash
echo "Hello, DevOps!"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(520).png)

What happens if we remove the shebang line?<br>
If we remove the shebang `#!/bin/bash`, the script may still run if we explicitly call it with bash `hello.sh`. But if we run `./hello.sh`, the system uses the default shell (often `/bin/sh`), which might not support all Bash features.

## Task 2: Variables
`vim variables.sh`

```bash
#!/bin/bash
NAME="Atul"
ROLE="DevOps Engineer"

echo "Hello, I am $NAME and I am a $ROLE"
echo 'Hello, I am $NAME and I am a $ROLE'
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(521).png)

- Double quotes expand variables.

- Single quotes treat everything literally.

## Task 3: User Input with read
`vim greet.sh`

```bash
#!/bin/bash
read -p "Enter your name: " NAME
read -p "Enter your favourite tool: " TOOL

echo "Hello $NAME, your favourite tool is $TOOL"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(522).png)

## Task 4: If-Else Conditions
`vim check_number.sh`

```bash
#!/bin/bash
read -p "Enter a number: " NUM

if [ $NUM -gt 0 ]; then
    echo "Positive"
elif [ $NUM -lt 0 ]; then
    echo "Negative"
else
    echo "Zero"
fi
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(523).png)

`vim file_check.sh`

```bash
#!/bin/bash
read -p "Enter filename: " FILE

if [ -f "$FILE" ]; then
    echo "File exists."
else
    echo "File does not exist."
fi
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(524).png)

## Task 5: Combine It All
`vim server_check.sh`

```bash
#!/bin/bash
SERVICE="nginx"

read -p "Do you want to check the status? (y/n): " ANSWER

if [ "$ANSWER" == "y" ]; then
    systemctl status $SERVICE > /dev/null 2>&1
    if [ $? -eq 0 ]; then
        echo "$SERVICE is active."
    else
        echo "$SERVICE is not active."
    fi
else
    echo "Skipped."
fi
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f97f21edda521a6c7e36dcf18b54f331d4bf7430/2026/day-16/Screenshots/Screenshot%20(525).png)

## Key Learnings
1. Shebang matters — it defines which interpreter runs your script.

2. Variables and quotes — double quotes expand variables, single quotes don’t.

3. Control flow — `if-else` lets us handle conditions, making scripts interactive and dynamic.