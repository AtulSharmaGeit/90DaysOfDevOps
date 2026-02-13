# File I/O Practice

## Goal
Practice basic file creation, writing, appending, and reading using fundamental commands.

---

## Commands and Explanations

1. `touch notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(442).png)

   - Creates an empty file named *notes.txt*.

2. `echo "Line 1" > notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(443).png)

   - Writes "Line 1" into the file (overwrites if file already has content).

3. `echo "Line 2" >> notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(444).png)

   - Appends "Line 2" to the file.

4. `echo "Line 3" | tee -a notes.txt` 

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(445).png)

   - Appends "Line 3" to the file **and** displays it on screen.

5. `cat notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(446).png)

   - Reads and displays the full file contents.

6. `head -n 2 notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(448).png)

   - Displays the first 2 lines of the file.

7. `tail -n 2 notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(447).png)

   - Displays the last 2 lines of the file.

8. `rm notes.txt`  

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/58cfe7aa6b5e82288bae8341cf46e20084ec7de3/2026/day-06/Images/Screenshot%20(449).png)

   - Permanently remove files and directories from the filesystem.

---

