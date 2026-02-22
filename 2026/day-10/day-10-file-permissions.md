# File Permissions & File Operations.

## Files Created
- Create empty file

    `touch devops.txt`

- Create file with content

    `echo "These are my DevOps notes" > notes.txt`

- Create script using vim

    `vim script.sh`
    
- Inside vim, add:

    `echo "Hello DevOps"`
    
    Save and exit (:wq)

- Verify files

    `ls -l`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(476).png)

- Read notes.txt

    `cat notes.txt`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(477).png)

- Open script.sh in read-only mode

    `vim -R script.sh`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(479).png)

- Display first 5 lines of /etc/passwd

    `head -n 5 /etc/passwd`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(480).png)

- Display last 5 lines of /etc/passwd

    `tail -n 5 /etc/passwd`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(481).png)

## Permission Changes
- Format: rwxrwxrwx → owner, group, others

- Values: r=4, w=2, x=1

- Example output:

    `ls -l devops.txt notes.txt script.sh`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(482).png)

    Answer:

    - devops.txt → -rw-rw-r-- → owner can read/write, group can read/write, others can read.

    - notes.txt → -rw-rw-r-- → same as above.

    - script.sh → -rw-rw-r-- → same as above.

- Make script executable

    `chmod +x script.sh`

    `./script.sh   # Runs "Hello DevOps"`

- Set devops.txt to read-only

    `chmod a-w devops.txt`

- Set notes.txt to 640 (rw for owner, r for group, none for others)

    `chmod 640 notes.txt`

- Create directory with 755 permissions

    `mkdir project`

    `chmod 755 project`

- Verify after each change

    `ls -l`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(483).png)

- Try writing to read-only file

    `echo "new text" >> devops.txt`

     Error: "Permission denied"

     ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(484).png)

- Remove execute permission and try running script

    `chmod -x script.sh`

    `./script.sh`
    
    Error: "Permission denied"

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5ca790e18af2cce25c12dc784a7825b6fcfa8fe9/2026/day-10/Images/Screenshot%20(485).png)

## What I Learned
1. *File Operations* : I practiced creating, editing, and reading files using *touch*, *echo*, *cat*, and *vim*, and explored system files with *head* and *tail*.

2. *Permissions Fundamentals* : I understood the *rwxrwxrwx* format, numeric values (r=4, w=2, x=1), and how ownership affects read/write/execute access.

3. *Permission Management* : I modified permissions with *chmod* (symbolic and numeric), tested restrictions, and observed how Linux enforces security through error messages like “Permission denied.” 