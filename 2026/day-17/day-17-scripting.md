# SHELL SCRIPTING: LOOPS, ARGUMENTS & ERROR HANDLING

## Task 1: For Loop
`vim for_loop.sh`

```bash
#!/bin/bash
# Loop through a list of fruits
for fruit in apple banana mango orange grape
do
  echo "Fruit: $fruit"
done
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(526).png)

`vim count.sh`

```bash
#!/bin/bash
# Print numbers 1 to 10
for i in {1..10}
do
  echo $i
done
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(527).png)

## Task 2: While Loop
`vim countdown.sh`

```bash
#!/bin/bash
# Countdown from user input
read -p "Enter a number: " NUM
while [ $NUM -ge 0 ]
do
  echo $NUM
  NUM=$((NUM-1))
done
echo "Done!"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(528).png)

## Task 3: Command-Line Arguments
`vim greet.sh`

```bash
#!/bin/bash
# Greet user with argument
if [ $# -eq 0 ]; then
  echo "Usage: ./greet.sh <name>"
  exit 1
fi
echo "Hello, $1!"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(529).png)

`vim args_demo.sh`

```bash
#!/bin/bash
# Show arguments info
echo "Total arguments: $#"
echo "All arguments: $@"
echo "Script name: $0"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(530).png)

## Task 4: Install Packages via Script
`vim install_packages.sh`

```bash
#!/bin/bash
# Install packages if missing

# Check if running as root
if [ "$EUID" -ne 0 ]; then
  echo "Run as root"
  exit 1
fi

packages=(nginx curl wget)

for pkg in "${packages[@]}"
do
  if dpkg -s $pkg &> /dev/null; then
    echo "$pkg is already installed"
  else
    echo "Installing $pkg..."
    apt-get install -y $pkg || echo "Failed to install $pkg"
  fi
done
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(531).png)

## Task 5: Error Handling
`vim safe_script.sh`

```bash
#!/bin/bash
set -e

mkdir /tmp/devops-test || echo "Directory already exists"
cd /tmp/devops-test || echo "Failed to enter directory"
touch testfile.txt || echo "Failed to create file"
echo "Script completed successfully"
```

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/374d8b689a8b7c13befb91f2aa0182621c02807c/2026/day-17/Screenshots/Screenshot%20(532).png)

## Key Learnings
1. Loops (for, while) let you automate repetitive tasks easily.

2. Arguments ($1, $#, $@) make scripts dynamic and reusable.

3. Error handling (set -e, ||) ensures scripts fail gracefully instead of breaking silently.