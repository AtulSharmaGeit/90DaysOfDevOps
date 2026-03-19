
# INTRODUCTION TO DOCKER.

## TASK-1: WHAT IS DOCKER?
1. What is a container and why do we need them?<br>
**Ans.** 
    - A container is a lightweight, portable unit that packages an application and its dependencies together.
    - It ensures consistency across environments (dev, test, prod).
    - Unlike traditional deployments, containers isolate apps from the host OS, reducing conflicts.
2. Containers vs Virtual Machines — what's the real difference?<br>
**Ans.**
    |Feature   |	Containers          |	Virtual Machines              |
    |----------|------------------------|---------------------------------|
    |Isolation |	Share host OS kernel|	Full OS per VM                |
    |Size      |	MBs                 |	GBs                           |
    |Startup   |	Seconds             |	Minutes                       |
    |Efficiency|	Lightweight, faster |	Heavy, resource-intensive     |
    |Use Case  |	Microservices, CI/CD|	Legacy apps, full OS isolation|
3. What is the Docker architecture?<br>
**Ans.**
    - **Docker Daemon:** Background service managing images, containers, networks, volumes.
    - **Docker Client:** CLI (`docker run`, `docker ps`) that talks to the daemon.
    - **Images:** Read-only templates used to create containers.
    - **Containers:** Running instances of images.
    - **Registry:** Central store (e.g., Docker Hub) for images.

    Docker client sends commands to the Docker daemon. The daemon pulls images from a registry (like Docker Hub), creates containers from those images, and manages their lifecycle.


## TASK-2: INSTALL DOCKER
1. On Linux (Ubuntu):
    ```bash
    sudo apt update
    sudo apt install docker.io -y
    sudo systemctl start docker
    sudo systemctl enable docker
    sudo systemctl status docker
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(57).png)

2. Verify installation:
    ```bash
    docker --version
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(58).png)

3. Add Our User to the docker Group<br>
    To run Docker without `sudo`, add yourself to the `docker` group
    ```bash
    sudo groupadd docker
    sudo usermod -aG docker $USER
    ```
4. Run Hello World
    ```bash
    sudo docker run hello-world
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(59).png)

Docker pulled the image from Docker Hub, created a container, ran it, and displayed the message.

## TASK-3: RUN REAL CONTAINERS
1. Run Nginx
    ```bash
    sudo docker run -d -p 8081:80 nginx
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(60).png)

    - Visit http://localhost:8081 in our browser
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(62).png)

    - Nginx welcome page.
2. Run Ubuntu Interactive
    ```bash
    sudo docker run -it ubuntu bash
    ```
    Explore with ls, pwd, apt update.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(63).png)

3. List Containers
    ```bash
    sudo docker ps        # running containers
    sudo docker ps -a     # all containers
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(64).png)

4. Stop & Remove
    ```bash
    sudo docker stop <container_id>
    sudo docker rm <container_id>
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(67).png)

## TASK-4: EXPLORE
1. Run a Container in Detached Mode
    ```bash
    sudo docker run -d ubuntu sleep 100
    ```
    - `-d` - detached mode (runs in background).
    - We won’t see the container’s output in our terminal.
    - Verify with:
        ```bash
        sudo docker ps
        ```
        - Shows the container running.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(68).png)

2. Give a Container a Custom Name
    ```bash
    sudo docker run -d --name myubuntu ubuntu sleep 100
    ```
    - `--name myubuntu` - assigns a readable name.
    - Easier to manage than random IDs.
    - Stop/remove by name:
    ```bash
    sudo docker stop myubuntu
    sudo docker rm myubuntu
    ```
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(70).png).

3. Map a Port from the Container to Your Host
    ```bash
    sudo docker run -d -p 8081:80 nginx
    ```
    - Maps host port 8080 - container port 80.
    - Access via http://localhost:8080 in our browser.
    - Without port mapping, we can’t reach the service externally.
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(62).png)

4. Check Logs of a Running Container
    ```bash
    docker logs <container_name_or_id>
    ```
    - Displays container output (startup messages, errors, etc.).
    - Useful for debugging.

    ![iamge alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(73).png)

5. Run a Command Inside a Running Container
    ```bash
    docker exec -it myubuntu 
    ```
    - `exec` → run a command inside an existing container.
    - `-it` → interactive terminal.

    We now have a shell inside the container.
    ```bash
    ls
    cat /etc/os-release
    ```
    Exit with:
    ```bash
    exit
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/5d91862f3965ac3b3a24995c3418180187b0addd/2026/day-29/Screenshots/Screenshot%20(76).png)


