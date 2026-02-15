# DEPLOYING A WEB SERVER ON AWS CLOUD

## Commands Used
1. SSH into Instance

    `ssh -i "90DaysOfDevOps.pem" ubuntu@ec2-13-232-151-252.ap-south-1.compute.amazonaws.com`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e08c3fc755d6afc90105699429448ad82c072b03/2026/day-08/Images/Screenshot%20(460).png)

2. Update system & install Nginx

    `sudo apt update && sudo apt upgrade -y`

    `sudo apt install nginx -y`
    
    `sudo systemctl start nginx`

    `sudo systemctl enable nginx`

    `sudo systemctl status nginx`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e08c3fc755d6afc90105699429448ad82c072b03/2026/day-08/Images/Screenshot%20(461).png)

3. Test in browser

- Edit EC2 Security Group → Add inbound rule:
    - HTTP (80) → Source: 0.0.0.0/0 (for public access)

        `http://13.232.151.252`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e08c3fc755d6afc90105699429448ad82c072b03/2026/day-08/Images/Screenshot%20(462).png)

4. View logs

    `sudo cat /var/log/nginx/access.log`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e08c3fc755d6afc90105699429448ad82c072b03/2026/day-08/Images/Screenshot%20(463').png)
    
    `sudo cat /var/log/nginx/error.log`

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/e08c3fc755d6afc90105699429448ad82c072b03/2026/day-08/Images/Screenshot%20(464).png)

5. Save logs to file    
    
    `sudo cp /var/log/nginx/access.log ~/nginx-logs.txt`

6. Download logs to local machine

    `scp -i 90DaysOfDevOps.pem ubuntu@13.232.151.252:~/nginx-logs.txt .`

## Challenges Faced
- SSH connection failed due to wrong key permissions → solved with `chmod 400 90DaysOfDevOps.pem`
- Browser not loading Nginx → solved by adding inbound rule for port 80 in Security Group

## What I Learned
- How to launch and connect to a cloud instance
- Installing and managing services (Nginx) on Linux
- Configuring AWS Security Groups for web access
- Extracting and transferring server logs
- Verifying real-world deployment with browser access
