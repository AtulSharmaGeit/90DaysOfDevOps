# DOCKERFILE : BUILD YOUR OWN IMAGES

## Task 1: Our First Dockerfile
1. Create a folder called `my-first-image`
    ```bash
    mkdir my-first-image
    cd my-first-image/
    ```
2. Inside it, create a `Dockerfile`
    ```bash
    vim Dockerfile
    ```
    ```dockerfile
    FROM ubuntu:latest
    RUN apt-get update && apt-get install -y curl
    CMD ["echo", "Hello from my custom image!"]
    ```
3. Build & Run:
    ```bash
    docker build -t my-ubuntu:v1 .
    docker run my-ubuntu:v1
    ```
![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(104).png)

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(103).png)

## Task 2: Dockerfile Instructions
```dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl
COPY hello.txt /app/hello.txt
WORKDIR /app
EXPOSE 8080
CMD ["cat", "hello.txt"]
```
- **FROM:** Base image (Ubuntu).
- **RUN:** Executes commands during build.
- **COPY:** Copies files from host ---> image.
- **WORKDIR:** Sets working directory inside container.
- **EXPOSE:** Documents port (not enforced).
- **CMD:** Default command when container runs.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(105).png)

## Task 3: CMD vs ENTRYPOINT
1. CMD Example
    ```dockerfile
    FROM ubuntu:latest
    CMD ["echo", "hello"]
    ```
    ```bash
    docker build -t my-cmd:v1
    docker run my-cmd:v1
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(106).png)

    - Prints `hello`
    ```bash
    docker run my-cmd:v1 ls
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(107).png)

    - Docker replaced the CMD with `ls`, so instead of echoing, it listed the container’s filesystem.
    - CMD provides a default command, but it can be overridden at runtime.
2. ENTRYPOINT Example
    ```dockerfile
    FROM ubuntu:latest
    ENTRYPOINT ["echo"]
    ```
    ```bash
    docker build -t my-entry:v1
    docker run my-entry:v1 
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(108).png)

    - Prints nothing (needs args).
    ```bash
    docker run my-entry:v1 hello 
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(109).png)

    - Prints hello.
- **CMD** = default arguments, can be overridden.
- **ENTRYPOINT** = fixed executable, arguments are appended.
- Use **CMD** when we want a flexible default that can be overridden.
- Use **ENTRYPOINT** when we want a fixed executable and only allow arguments to change.

## Task 4: Build a Simple Web App Image
1. Create a small static HTML file 
    ```html
    <!DOCTYPE html>
    <html>
    <head><title>My Website</title></head>
    <body><h1>Hello from Dockerized Nginx!</h1></body>
    </html>
    ```
2. Dockerfile
    ```dockerfile
    FROM nginx:alpine
    COPY index.html /usr/share/nginx/html/index.html
    ```
3. Build & Run:
    ```bash
    docker build -t my-website:v1 .
    docker run -d -p 8081:80 my-website:v1
    ```
    Open browser → http://localhost:8081

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(110).png)

## Task 5: .dockerignore
1. Create a Project Folder
    ```bash
    mkdir dockerignore-demo
    cd dockerignore-demo
    ```
2. Make a file so we have something to copy into the image.
    ```bash
    echo "Hello Docker!" > hello.txt
    ```
3. Create a Dockerfile
    ```bash
    vim Dockerfile
    ```
    ```dockerfile
    FROM ubuntu:latest
    WORKDIR /app
    COPY . .
    CMD ["cat", "hello.txt"]
    ```
4. Create a `.dockerignore` File in the same folder.
    ```bash
    vim .dockerignore
    ```
    ```Code
    node_modules
    .git
    *.md
    .env
    ```
5. Add Some “Junk” Files to Test
    ```bash
    mkdir node_modules
    echo "junk code" > node_modules/test.js

    mkdir .git
    echo "fake git data" > .git/config

    echo "notes" > notes.md
    echo "SECRET_KEY=12345" > .env
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(111).png)

    - Now our folder has:
        - hello.txt (important file)
        - node_modules/ (junk)
        - .git/ (junk)
        - notes.md (junk)
        - .env (junk)

6. Build the Image
    ```bash
    docker build -t dockerignore-test:v1 .
    ```
    - Docker will skip the files listed in .dockerignore.

7. Run the Container
    ```bash
    docker run dockerignore-test:v1
    ```
    - Output: Hello Docker!

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(112).png)

8. Verify Ignored Files<br>
    Open a shell inside the container:
    ```bash
    docker run -it dockerignore-test:v1 bash
    ```
    Check the /app folder:
    ```bash
    ls /app
    ```

    ![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(114).png)

    We should only see `hello.txt`. No node_modules, no .git, no .md, no .env.

## Task 6: Build Optimization
1. Create a Simple Project
    ```bash
    mkdir build-demo
    cd build-demo
    ```
    Add a file:
    ```bash
    echo 'console.log("Hello Docker!")' > app.js
    ```
2. Write the First Dockerfile
    ```dockerfile
    FROM node:18
    WORKDIR /app
    COPY package.json .
    RUN npm install
    COPY . .
    CMD ["node", "app.js"]
    ```
3. Build the Image
    ```bash
    docker build -t build-demo:v1 .
    ```
    Docker will create **layers** for each instruction.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(124).png)

4. Change One Line<br>
    Edit `app.js`:
    ```js
    console.log("Hello Again!");
    ```
    Rebuild:
    ```bash
    docker build -t build-demo:v2 .
    ```
    - Docker **uses cache** for all steps until `COPY . .` because only the source code changed.
    - Only the last layers rebuild.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/567c61065f91e5295e836ad6469f6beec8c9943f/2026/day-31/Screenshots/Screenshot%20(125).png)

5. Why Order Matters<br>
    If we wrote the Dockerfile like this:
    ```dockerfile
    FROM node:18
    WORKDIR /app
    COPY . .
    RUN npm install
    CMD ["node", "app.js"]
    ```
    - Now when you change `app.js`, the `COPY . .` step invalidates the cache **before** `RUN npm install`.
    - Docker reruns `npm install` every time - slower builds.

6. Optimized Order<br>
    Best practice is to put **stable steps first** and **frequently changing steps last**:
    ```dockerfile
    FROM node:18
    WORKDIR /app
    COPY package.json .
    RUN npm install
    COPY . .
    CMD ["node", "app.js"]
    ```
    - `package.json` changes rarely - cache stays valid.
    - `COPY . .` changes often - placed last so only app code rebuilds.

    Why does layer order matter for build speed?<br>
    **Ans.** Layer order matters because Docker builds images in **layers**, and if one layer changes, all the layers after it must rebuild. By putting **rarely changing steps first** and **frequently changing steps last**, Docker can reuse cached layers and rebuild only what’s necessary, making builds much faster.