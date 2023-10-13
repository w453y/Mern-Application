# Dockerizing Mern-Application


## Table of Contents
- [Cloning the Repository](#cloning-the-repository)
- [Building the Docker Image](#building-the-docker-image)
- [Pushing Image to DockerHub](#pushing-image-to-dockerhub)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Github Workflow](#github-workflow)

## Cloning the Repository
```bash
git clone https://github.com/puranikvinit/mern-application.git
```
## Building the Docker Image

1. Created a Dockerfile with the configuration given below to build an image and run it on container.
```bash
FROM node:14.21.3-alpine

WORKDIR /app
COPY ./package.json package.json
COPY ./package-lock.json package-lock.json
COPY ./.env .env
COPY ./start.sh start.sh
COPY ./backend backend/
COPY ./frontend frontend/
COPY ./screenshots screenshots/
RUN npm install && \
    cd frontend && \
    npm install && \
    cd .. && \
    chmod +x start.sh
CMD ["/app/start.sh"]
```
2. Created docker-compose file with the configuration given below to run both app and database on the same network.
```bash
version: "3.1"

services:

  app:
    build:
      context: .
    ports:
      - 5173:5173
      - 5000:5000
    expose:
      - "5000"
    depends_on:
      - db

  db:
    image: mongo:7.0.2-jammy
    restart: always
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=password
    ports:
      - "27017:27017"
    expose:
      - "27017"
```
3. As backend and frontend running on same container ( can't specify two entry point for the container ), so created a script which start backend and frontend service in the container with below configuration and added script as a entry point for the container.
```bash
#!/bin/sh

cd /app && npm start &
cd /app/frontend && npm run dev
```
4. Now its time to run ```docker compose up --build``` and the containers are up, as shown below.

![app_container](./screenshots/png1.png?raw=true "app_container")

As the container is running but web page doesn't load in the browser, as shown below

![error_browser](./screenshots/png2.png?raw=true "error_browser")

So to fix this, added this line ``` "dev": "vite --host" ``` in frontend/package.json, again build the image and web-site works correctly as show below

![web_page1](./screenshots/png3.png?raw=true "web_page")

5. As mentioned in the task " Expose only the required ports " due to this made some changes in docker-compose file and map only frontend port (5173) with localhost and exposed other ports i.e backend (5000) & database (27017) as shown below.
```bash
version: "3.1"

services:

  app:
    build:
      context: .
    ports:
      - 5173:5173
    expose:
      - "5000"
    depends_on:
      - db

  db:
    image: mongo:7.0.2-jammy
    restart: always
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=password
    expose:
      - "27017"
```
After making changes the frontend is working well but whenever frontend wants to fetch files from backend it can't reach backend, this happed because the frontend is expecting backend runs on localhost:5000 ( by default in frontend code the backend uri is localhost:5000) so it try to send request on port 5000 of localhost. As shown below

![backend_error](./screenshots/png4.png?raw=true "backend_error")

To solve this error we can use socat to flood incoming traffic from localhost:5000 to <container_ip>:5000
```bash
socat TCP-LISTEN:5000,fork,reuseaddr TCP-CONNECT:<container_ip>:5000
```
![backend_solved](./screenshots/png5.png?raw=true "backend_solved")

## Pushing Image to DockerHub

1. Taging the image ``` docker tag mern-application-app:latest w453y/wec-task-containerization:latest ```
2. Push image to docker hub ``` docker push w453y/wec-task-containerization:latest ```

## Kubernetes Deployment

Created a manifest file with the below configurations.
```bash
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: db
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017

---

apiVersion: v1
kind: Service
metadata:
  name: website-service
spec:
  selector:
    app: app
  ports:
    - name: port1
      protocol: TCP
      port: 5173
      targetPort: 5173
    - name: port2
      protocol: TCP
      port: 5000
      targetPort: 5000
  type: NodePort

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: mongo:7.0.2-jammy
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: root
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: password

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: w453y/wec-task-containerization:latest
        ports:
        - containerPort: 5000
        - containerPort: 5173
```

## Github Workflow

Created .yml file for github actions and configured it as shown below.

```bash
name: Build and Push Docker Images to DockerHub

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Login to DockerHub
      run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and tag Docker images
      run: |
        docker build -t w453y/wec-task-containerization:latest .
        docker tag w453y/wec-task-containerization:latest w453y/wec-task-containerization:latest

    - name: Push Docker images
      run: |
        docker push w453y/wec-task-containerization:latest
```
Github Actions worked without any errors as show below

![actions](./screenshots/png6.png?raw=true "actions")
