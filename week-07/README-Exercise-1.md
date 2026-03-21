# Exercise 1 — Containerise the Taskline App - Docker

---

## Overview

In this exercise you will write a Dockerfile for the taskline app, build it into a container image, and run it locally. By the end, your app should be reachable at `http://localhost:3000` — not from Node running on your machine, but from inside a container.

---

## What You Need

- Docker Desktop installed and running
- The app source code here - https://....
- A terminal

---

## Your Task

### 1. Create a `.dockerignore` file

In the root of the project, create a file named `.dockerignore` with the following contents:

```
node_modules
.git
*.log
.env
```

This prevents your local `node_modules` from being copied into the image.

---

### 2. Create a `Dockerfile`

In the root of the project, create a file named `Dockerfile` (no extension) with the following:

```dockerfile
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

COPY . .
RUN npm run build

FROM node:20-alpine AS runtime

WORKDIR /app
ENV NODE_ENV=production

COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts

COPY --from=build /app/dist ./dist
COPY --from=build /app/server.js ./server.js

EXPOSE 8080

CMD ["npm", "start"]

```

---

### 3. Build the image

From the project root directory, run:

```bash
docker build -t tasklineapp:local .
```

You should see Docker executing each step. The final line should read **Successfully built**.

Verify the image exists:

```bash
docker images
```

You should see `tasklineapp` with tag `local` in the list.

---

### 4. Run the container

```
docker run --rm --name my-taskline-app -p 3000:8080 tasklineapp:local
```

Open your browser and go to:

```
http://localhost:3000
```

The taskline app should load and be fully playable.

---

### 5. Stop the container

Press `Ctrl+C` in the terminal to stop it.

---

## Submitting Your Work

Push the following to your GitHub repository and submit the screenshots

---

## Hints

- Make sure Docker Desktop is running before you use any `docker` command — you'll get a connection error if the daemon isn't up.
- If you see `port is already allocated`, something else is using port 3000. Either stop that process, or change the host port: `docker run -p 3001:3000 tasklineapp:local` and visit `localhost:3001`.
- If `npm install` fails during the build, check that your `package.json` is valid JSON and all dependency names are spelled correctly.
- Run `docker ps` to see containers that are currently running. Run `docker ps -a` to see all containers including stopped ones.
