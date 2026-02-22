# USER AND GROUP MANAGEMENT

## Users & Groups Created
- Users: tokyo, berlin, professor, nairobi

    `sudo useradd -m tokyo` 

    `sudo useradd -m berlin` 

    `sudo useradd -m professor`

- Set passwords

    `sudo passwd tokyo`
    
    `sudo passwd berlin`
    
    `sudo passwd professor`

- Verify users 

    `cat /etc/passwd | grep -E "tokyo|berlin|professor"` 
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(465).png)

    `ls /home/`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(466).png)

- Groups: developers, admins, project-team
    
    `sudo groupadd developers` 

    `sudo groupadd admins`

- Verify groups 

    `cat /etc/group | grep -E "developers|admins"`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(467).png)

## Group Assignments
- tokyo → developers

    `sudo usermod -aG developers tokyo`

- berlin → developers + admins

    `sudo usermod -aG developers,admins berlin`

- professor → admins

    `sudo usermod -aG admins professor`

- Verify memberships
    
    `groups tokyo`

    `groups berlin`

    `groups professor`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(468).png)

## Directories Created
- Create directory

    `sudo mkdir /opt/dev-project`

- Set group owner

    `sudo chgrp developers /opt/dev-project`

- Set permissions

    `sudo chmod 775 /opt/dev-project`

- Verify

    `ls -ld /opt/dev-project`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(470).png)

- Test file creation

    `sudo -u tokyo touch /opt/dev-project/tokyo.txt`

    `sudo -u berlin touch /opt/dev-project/berlin.txt`

    `ls -l /opt/dev-project/`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(471).png)

## Team Workspace
-  Create user nairobi

    `sudo useradd -m nairobi`

    `sudo passwd nairobi`

- Create group project-team

    `sudo groupadd project-team`

- Add users to project-team

    `sudo usermod -aG project-team nairobi`

    `sudo usermod -aG project-team tokyo`

- Create directory

    `sudo mkdir /opt/team-workspace`

- Set group and permissions

    `sudo chgrp project-team /opt/team-workspace`

    `sudo chmod 775 /opt/team-workspace`

- Verify

    `ls -ld /opt/team-workspace`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(474).png)

- Test file creation

    `sudo -u nairobi touch /opt/team-workspace/nairobi.txt`

    `ls -l /opt/team-workspace/`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/3345cf7480595cdabfa30b14e98203132cbba9b7/2026/day-09/Images/Screenshot%20(475).png)

## What I Learned
- How to create and manage users with home directories and passwords.

- How to create groups and assign users to multiple groups.

- How to configure shared directories with group ownership and permissions for collaboration.