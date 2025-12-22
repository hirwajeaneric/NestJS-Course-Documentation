   # Module 13: Docker Fundamentals

## 📚 Table of Contents
1. [Overview](#overview)
2. [What is Docker?](#what-is-docker)
3. [Docker Installation](#docker-installation)
4. [Docker Concepts](#docker-concepts)
5. [Creating a Dockerfile](#creating-a-dockerfile)
6. [Building Docker Images](#building-docker-images)
7. [Running Containers](#running-containers)
8. [Docker Commands](#docker-commands)
9. [Multi-stage Builds](#multi-stage-builds)
10. [Docker Networking](#docker-networking)
11. [Volumes and Data Persistence](#volumes-and-data-persistence)
12. [Best Practices](#best-practices)
13. [Mini Project: Containerize NestJS App](#mini-project-containerize-nestjs-app)

---

## Overview

Docker is a platform for developing, shipping, and running applications in containers. This module teaches you how to containerize your NestJS applications for consistent deployment across different environments.

**Why Docker?**
- **Consistency**: Works the same on dev, staging, and production
- **Isolation**: Containers don't interfere with each other
- **Portability**: Run anywhere Docker is installed
- **Scalability**: Easy to scale horizontally
- **Dependency Management**: All dependencies bundled in the container

---

## What is Docker?

### Containers vs Virtual Machines

**Containers:**
- Share the host OS kernel
- Lightweight and fast to start
- Use less resources
- Isolated but share OS

**Virtual Machines:**
- Full OS for each VM
- Heavier and slower
- More resource intensive
- Complete isolation

### Docker Components

- **Dockerfile**: Instructions to build an image
- **Image**: Template for creating containers
- **Container**: Running instance of an image
- **Registry**: Storage for images (Docker Hub, ECR, etc.)

---

## Docker Installation

### macOS

```bash
# Install Docker Desktop (includes Docker CLI and GUI)
brew install --cask docker

# Or download from: https://www.docker.com/products/docker-desktop
```

### Linux (Ubuntu/Debian)

```bash
# Update package index
sudo apt update

# Install prerequisites
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (to run without sudo)
sudo usermod -aG docker $USER
```

### Windows

Download Docker Desktop from: https://www.docker.com/products/docker-desktop

### Verify Installation

```bash
# Check Docker version
docker --version

# Test Docker installation
docker run hello-world
```

---

## Docker Concepts

### Image

An **image** is a read-only template with instructions for creating a container. Images are built from Dockerfiles.

```bash
# List local images
docker images

# Pull an image from Docker Hub
docker pull node:18-alpine

# Remove an image
docker rmi node:18-alpine
```

### Container

A **container** is a running instance of an image. Multiple containers can run from the same image.

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Start a container
docker start <container-id>

# Stop a container
docker stop <container-id>

# Remove a container
docker rm <container-id>
```

---

## Creating a Dockerfile

A Dockerfile contains instructions for building a Docker image. Here's how to create one for a NestJS application:

### Basic Dockerfile

```dockerfile
# Dockerfile for NestJS Application

# Step 1: Use Node.js 18 Alpine as base image
# Alpine Linux is minimal and secure (smaller image size)
# Official Node.js image maintained by Node.js team
FROM node:18-alpine

# Step 2: Set working directory inside container
# All subsequent commands run in this directory
# Note: /app is a path INSIDE the container, not on your host machine
# Your NestJS project structure (src/, test/, node_modules/) will be copied here
# You can use any path like /usr/src/app, /opt/app, or /app - it's just a convention
WORKDIR /app

# Step 3: Copy package files
# Copy package.json and package-lock.json (or yarn.lock)
# This is done before copying source code to leverage Docker layer caching
# If source code changes but dependencies don't, Docker will reuse cached layer
COPY package*.json ./

# Step 4: Install dependencies
# Install all npm packages defined in package.json
# --only=production installs only production dependencies (skips devDependencies)
RUN npm ci --only=production

# Step 5: Copy application source code
# Copy all files from current directory (.) to working directory (/app)
# .dockerignore file (similar to .gitignore) can exclude unnecessary files
COPY . .

# Step 6: Build the application
# Compile TypeScript to JavaScript
# This creates the dist/ directory with compiled code
RUN npm run build

# Step 7: Expose port
# Document which port the application listens on
# This doesn't actually publish the port (use -p flag when running)
EXPOSE 3000

# Step 8: Set environment variable
# Define the Node environment
ENV NODE_ENV=production

# Step 9: Define the command to run when container starts
# Start the production server
# CMD is the default command (can be overridden when running container)
CMD ["node", "dist/main.js"]
```

**Dockerfile Instructions Explained:**
- `FROM`: Sets the base image
- `WORKDIR`: Sets the working directory
- `COPY`: Copies files from host to container
- `RUN`: Executes a command during image build
- `EXPOSE`: Documents which port the app uses
- `ENV`: Sets environment variables
- `CMD`: Default command to run when container starts

**Important Note about WORKDIR:**
The `WORKDIR /app` instruction creates a directory `/app` **inside the Docker container**, not on your host machine. This works perfectly fine with NestJS projects that have the typical structure:
- `src/` - your source code
- `test/` - your test files
- `node_modules/` - dependencies
- `package.json` - project configuration

When you run `COPY . .`, Docker copies everything from your host directory (including `src/`, `test/`, etc.) into `/app` inside the container. The `/app` path is just a convention - you could use `/usr/src/app`, `/opt/app`, or any other path. What matters is that it's consistent throughout your Dockerfile.

### .dockerignore File

Create a `.dockerignore` file to exclude unnecessary files from the Docker build context:

```dockerignore
# .dockerignore

# Dependencies
node_modules
npm-debug.log
yarn-error.log

# Build output
dist
build

# Development files
.env.local
.env.development
.env.test

# Git
.git
.gitignore

# IDE
.vscode
.idea
*.swp
*.swo

# Documentation
README.md
*.md

# Tests
test
*.spec.ts
*.test.ts

# Docker
Dockerfile
.dockerignore
docker-compose.yml
```

**Why .dockerignore?**
- Reduces build context size (faster builds)
- Prevents sensitive files from being copied
- Excludes unnecessary files from the image

---

## Building Docker Images

Build a Docker image from a Dockerfile:

```bash
# Basic build command
# -t: Tag the image with a name:tag
# . : Build context (current directory)
docker build -t my-nestjs-app:latest .

# Build with specific tag
docker build -t my-nestjs-app:v1.0.0 .

# Build with no cache (forces rebuilding all layers)
docker build --no-cache -t my-nestjs-app:latest .

# Build with build arguments
docker build --build-arg NODE_ENV=production -t my-nestjs-app:latest .
```

**Understanding the Build Process:**
1. Docker reads the Dockerfile
2. Executes each instruction as a layer
3. Caches layers for faster subsequent builds
4. Creates a final image

**Layer Caching:**
- Each instruction creates a layer
- Unchanged layers are reused from cache
- Order matters: put frequently changing instructions last

---

## Running Containers

### Basic Container Run

```bash
# Run a container from an image
# docker run [options] image [command]
docker run my-nestjs-app:latest

# Run in detached mode (background)
# -d: Detached mode (runs in background)
docker run -d my-nestjs-app:latest

# Run with port mapping
# -p host-port:container-port maps ports
# Access app at http://localhost:3000
docker run -p 3000:3000 my-nestjs-app:latest

# Run with name (easier to reference)
# --name: Assign a name to the container
docker run --name my-app -p 3000:3000 my-nestjs-app:latest

# Run with environment variables
# -e: Set environment variables
docker run -e NODE_ENV=production -e PORT=3000 my-nestjs-app:latest

# Run with .env file
# --env-file: Load environment variables from file
docker run --env-file .env my-nestjs-app:latest
```

### Common Run Options

```bash
# Interactive mode with TTY (for debugging)
docker run -it my-nestjs-app:latest sh

# Remove container automatically when it stops
docker run --rm my-nestjs-app:latest

# Run with resource limits
docker run --memory="512m" --cpus="1.0" my-nestjs-app:latest

# Mount a volume (persistent data)
docker run -v /host/path:/container/path my-nestjs-app:latest

# Run with network
docker run --network my-network my-nestjs-app:latest
```

**Port Mapping Explained:**
- `-p 3000:3000`: Maps host port 3000 to container port 3000
- `-p 8080:3000`: Maps host port 8080 to container port 3000 (access at localhost:8080)

---

## Docker Commands

### Image Management

```bash
# List all images
docker images

# Remove an image
docker rmi <image-id>

# Remove all unused images
docker image prune

# Inspect an image
docker inspect <image-id>

# Tag an image
docker tag my-app:latest my-app:v1.0.0
```

### Container Management

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Start a stopped container
docker start <container-id>

# Stop a running container
docker stop <container-id>

# Restart a container
docker restart <container-id>

# Remove a container
docker rm <container-id>

# Remove all stopped containers
docker container prune

# View container logs
docker logs <container-id>

# Follow logs (like tail -f)
docker logs -f <container-id>

# Execute command in running container
docker exec -it <container-id> sh

# View container stats (CPU, memory)
docker stats <container-id>
```

### System Management

```bash
# System information
docker info

# Disk usage
docker system df

# Clean up everything (use with caution!)
docker system prune -a
```

---

## Multi-stage Builds

Multi-stage builds allow you to use multiple FROM statements in one Dockerfile, enabling you to create smaller final images by discarding intermediate build artifacts.

### Why Multi-stage Builds?

**Problem with Single-stage:**
- Build tools (TypeScript compiler, etc.) end up in final image
- Larger image size
- Security concerns (unnecessary tools)

**Solution: Multi-stage:**
- Build in one stage with all tools
- Copy only necessary files to final stage
- Final image contains only runtime dependencies

### Multi-stage Dockerfile Example

```dockerfile
# Stage 1: Build stage
# This stage has all build tools and dependencies
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies for building)
# This is different from production stage
RUN npm ci

# Copy source code
COPY . .

# Build the application
# This compiles TypeScript to JavaScript
RUN npm run build

# Stage 2: Production stage
# This stage has only runtime dependencies
FROM node:18-alpine AS production

# Set working directory
WORKDIR /app

# Copy package files again
COPY package*.json ./

# Install ONLY production dependencies
# No TypeScript, no build tools, smaller image
RUN npm ci --only=production && npm cache clean --force

# Copy built application from builder stage
# Only the dist/ directory is needed (not src/, not node_modules)
COPY --from=builder /app/dist ./dist

# Create non-root user for security
# Running as root is a security risk
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

# Change ownership of app directory
USER nestjs

# Expose port
EXPOSE 3000

# Set environment
ENV NODE_ENV=production

# Health check
# Docker can check if container is healthy
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/main.js"]
```

**Multi-stage Benefits:**
- **Smaller final image**: Only production dependencies
- **Security**: No build tools in production image
- **Faster deployments**: Smaller images transfer faster
- **Better caching**: Build and runtime stages cached separately

---

## Docker Networking

Docker creates virtual networks for containers to communicate.

### Default Networks

```bash
# List networks
docker network ls

# Inspect a network
docker network inspect bridge

# Create a custom network
docker network create my-network

# Connect container to network
docker network connect my-network <container-id>

# Disconnect container from network
docker network disconnect my-network <container-id>
```

### Network Types

1. **Bridge**: Default network for containers on same host
2. **Host**: Uses host's network directly (Linux only)
3. **None**: No networking
4. **Overlay**: For multi-host networking (Swarm)

### Example: Connecting Containers

```bash
# Create a network
docker network create app-network

# Run database container on network
docker run -d --name postgres --network app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:14

# Run app container on same network
docker run -d --name my-app --network app-network \
  -e DATABASE_HOST=postgres \
  -p 3000:3000 \
  my-nestjs-app:latest

# Containers can now communicate using container names
# App can connect to database at: postgres:5432
```

---

## Volumes and Data Persistence

Containers are ephemeral - data is lost when container is removed. Volumes provide persistent storage.

### Volume Types

1. **Named Volumes**: Managed by Docker (recommended)
2. **Bind Mounts**: Map to host directory
3. **Anonymous Volumes**: Temporary volumes

### Using Volumes

```bash
# Create a named volume
docker volume create postgres-data

# List volumes
docker volumes ls

# Inspect volume
docker volume inspect postgres-data

# Run container with volume
docker run -d --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:14

# Run with bind mount (host directory)
docker run -d --name my-app \
  -v /host/path:/container/path \
  my-nestjs-app:latest
```

### Dockerfile Volume

```dockerfile
# Declare a volume in Dockerfile
VOLUME ["/app/uploads"]

# Data in /app/uploads persists even if container is removed
```

---

## Best Practices

### 1. Use .dockerignore

Always create a `.dockerignore` to exclude unnecessary files.

### 2. Use Specific Base Images

```dockerfile
# Good: Specific version
FROM node:18-alpine

# Bad: Latest tag (unpredictable)
FROM node:latest
```

### 3. Order Instructions Efficiently

Put frequently changing instructions last to leverage caching.

### 4. Minimize Layers

Combine RUN commands when possible:

```dockerfile
# Good: One layer
RUN npm ci && npm cache clean --force

# Bad: Multiple layers
RUN npm ci
RUN npm cache clean --force
```

### 5. Use Non-root User

```dockerfile
# Create and use non-root user
RUN adduser -D appuser
USER appuser
```

### 6. Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1
```

### 7. Multi-stage for Production

Always use multi-stage builds for production images.

### 8. Set Resource Limits

```bash
docker run --memory="512m" --cpus="1.0" my-app
```

---

## Mini Project: Containerize NestJS App

Containerize your NestJS application:

1. Create Dockerfile with multi-stage build
2. Create .dockerignore file
3. Build the image
4. Run the container
5. Test the application
6. Push to Docker Hub

### Step-by-Step:

```bash
# 1. Create Dockerfile (as shown above)

# 2. Create .dockerignore

# 3. Build image
docker build -t my-nestjs-app:latest .

# 4. Run container
docker run -d --name my-app -p 3000:3000 my-nestjs-app:latest

# 5. Test
curl http://localhost:3000

# 6. View logs
docker logs my-app

# 7. Stop and remove
docker stop my-app
docker rm my-app
```

---

## Next Steps

✅ Understand Docker concepts  
✅ Create Dockerfiles  
✅ Build and run containers  
✅ Use multi-stage builds  
✅ Understand networking and volumes  
✅ Move to Module 14: Docker Compose & Local Development  

---

*You now know how to containerize your applications with Docker! 🐳*

