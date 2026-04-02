# DOCKER COMPOSE: MULTI-CONTAINER BASICS.

## Task 1: Install & Verify

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(199).png)

## Task 2: My First Compose File
1. Create a Project Folder
    ```bash
    mkdir compose-basics
    cd compose-basics
    ```
2. Write the `docker-compose.yml`
    ```yaml
    version: "3.9"
    services:
      web:
        image: nginx:latest
        ports:
          - "8081:80"
    ```
    - **version:** Compose file format version.
    - **services:** Defines containers. Here we have one service called `web`.
    - **image:** Pulls the official Nginx image.
    - **ports:** Maps port `8081` on your machine - port `80` inside the container.
3. Start the Container
    ```bash
    docker-compose up -d
    ```
    - This will pull the Nginx image (if not already present).
    - It will start the container and attach logs to your terminal.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(200).png)

4. Access in Browser
    ```Code
    http://localhost:8081
    ```
    - We should see the default Nginx welcome page.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(201).png)

5. Stop and Clean Up
    ```bash
    docker-compose down
    ```
    - This stops and removes the container and network created by Compose.
    - Our YAML file remains, so we can start it again anytime with `docker compose up`.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(203).png)

## Task 3: Two-Container Setup
1. Create a Project Folder
    ```bash
    mkdir wordpress-compose
    cd wordpress-compose
    ```
2. Write the `docker-compose.yml`
    ```yaml
    services:
      db:
        image: mysql:8.0
        restart: always
        environment:
          MYSQL_DATABASE: wordpress
          MYSQL_USER: wpuser
          MYSQL_PASSWORD: wppassword
          MYSQL_ROOT_PASSWORD: rootpassword
        volumes:
          - db_data:/var/lib/mysql

      wordpress:
        image: wordpress:latest
        restart: always
        depends_on:
          - db
        ports:
          - "8081:80"
        environment:
          WORDPRESS_DB_HOST: db
          WORDPRESS_DB_USER: wpuser
          WORDPRESS_DB_PASSWORD: wppassword
          WORDPRESS_DB_NAME: wordpress

    volumes:
      db_data:
    ```
    - Key points:
        - Both services (`db` and `wordpress`) are automatically on the same network.
        - `db_data` is a named volume - ensures MySQL data persists.
        - WordPress connects to MySQL using the service name `db`.
3. Start the Containers
    ```bash
    docker-compose up -d
    ```
    - `-d` runs in detached mode.
    - Compose will pull images if not already present.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(218).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(220).png)

    ```bash
    docker-compose logs -f
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(221).png)

4. Access WordPress
    ```Code
    http://localhost:8081
    ```
    - You should see the WordPress setup page.
    - Complete the installation (site title, admin user, password, etc.).

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(224).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(225).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(227).png)

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(228).png)

5. Verify Persistence<br>
    Stop everything:
    ```bash
    docker-compose down
    ```
    - This removes containers and networks, but **not volumes**.
    
    Restart:
    ```bash
    docker-compose up -d
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(232).png)

    - Go back to `http://localhost:8081`  our WordPress site and data should still be there.

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/f1185685f17b3ab67a432d5bbf6ce8755508d1ad/2026/day-33/Screenshots/Screenshot%20(231).png)

    We now have a multi‑container application running with one command, and we've confirmed data persistence using a named volume.
## Task 4: Compose Commands
1. Start Services in Detached Mode<br>
    Run containers in the background:
    ```bash
    docker-compose up -d
    ```
    - `-d` = detached mode (no logs in your terminal).
    - Useful for long‑running services.
2. View Running Services<br>
    Check which services are up:
    ```bash
    docker-compose ps
    ```
    - Shows container names, status, and ports.
3. View Logs of All Services<br>
    See combined logs:
    ```bash
    docker-compose logs -f
    ```
    - `-f` = follow logs live (like tail -f).
4. View Logs of a Specific Service<br>
    Example for WordPress:
    ```bash
    docker-compose logs -f wordpress
    ```
    - Replace wordpress with the service name defined in our YAML.
5. Stop Services Without Removing<br>
    Pause containers but keep networks/volumes intact:
    ```bash
    docker-compose stop
    ```
    - Containers stop, but we can restart them quickly with:
        ```bash
        docker compose start
        ```
6. Remove Everything (Containers + Networks)<br>
    Clean up completely:
    ```bash
    docker-compose down
    ```
    - Removes containers and networks.
    - Named volumes persist unless we add -v:
        ```bash
        docker-compose down -v
        ```
7. Rebuild Images if We Make a Change<br>
    If we edit a Dockerfile or configuration:
    ```bash
    docker-compose up --build -d
    ```
    - Forces rebuild of images before starting.

## Task 5: Environment Variables
1. Add Environment Variables Directly in `docker-compose.yml`<br>
    Example:
    ```yaml
    version: "3.9"
    
    services:
      app:
        image: nginx:latest
        ports:
          - "8082:80"
        environment:
          APP_ENV: development
          APP_DEBUG: "true"
    ```
    - Here, `APP_ENV` and `APP_DEBUG` are passed directly into the container.
    - We can verify inside the container:
        ```bash
        docker exec -it <container_id> env
        ```
        - Look for `APP_ENV` and `APP_DEBUG`.

        ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(211).png)

2. Create a `.env` File<br>
    Create a file named `.env` in the same folder:
    ```Code
    APP_ENV=production
    APP_DEBUG=false
    ```
3. Reference Variables in `docker-compose.yml`<br>
    Update our Compose file:
    ```yaml
    version: "3.9"
    services:
    app:
        image: nginx:latest
        ports:
        - "8082:80"
        environment:
        APP_ENV: ${APP_ENV}
        APP_DEBUG: ${APP_DEBUG}
    ```
    - `${APP_ENV}` and `${APP_DEBUG}` will be replaced with values from `.env`.

4. Verify Variables Are Picked Up<br>
    Run:
    ```bash
    docker compose config
    ```
    - This command shows the fully resolved configuration.
    - We should see:
    
    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(213).png)

5. Run and Test<br>
    Start the container:
    ```bash
    docker compose up -d
    ```
    Check inside:
    ```bash
    docker exec -it <container_id> env
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/8c6ba2cc9d0e987c69b77cd238de4dcb595a8917/2026/day-33/Screenshots/Screenshot%20(214).png)

    - Confirms that the variables are set correctly.