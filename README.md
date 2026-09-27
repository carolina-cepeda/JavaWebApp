# Virtualization Lab: Containerizing and Deploying a Java Web Application

![Java](https://img.shields.io/badge/Java-21-blue.svg)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.1-brightgreen.svg)
![Maven](https://img.shields.io/badge/Maven-3.9+-red.svg)
![Docker](https://img.shields.io/badge/Docker-24.0+-blue.svg)
![Docker Compose](https://img.shields.io/badge/Docker%20Compose-v2-blue.svg)

## Overview

This repository contains the implementation of the **Virtualization Workshop**: building a Spring Boot web application, containerizing it with Docker, orchestrating with Docker Compose (including MongoDB), and publishing to Docker Hub.


---

## Table of Contents

- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Part 1: Spring Boot Application](#part-1-spring-boot-application)
- [Part 2: Docker Image](#part-2-docker-image)
- [Part 3: Docker Compose](#part-3-docker-compose)
- [Part 4: Docker Hub](#part-4-docker-hub)
- [Part 5: AWS EC2 Deployment](#part-5-aws-ec2-deployment)
- [Part 6: Cost Analysis](#part-6-cost-analysis)
- [How to Run](#how-to-run)
- [Evidence](#evidence)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Host                          │
│  ┌──────────────────┐         ┌──────────────────────────┐  │
│  │  Web Container   │         │      DB Container        │  │
│  │  (Spring Boot)   │◄───────►│      (MongoDB 7)         │  │
│  │  Port: 9000      │  db:27017│      Port: 27017         │  │
│  └────────┬─────────┘         └──────────────────────────┘  │
│           │                                                 │
│    ┌──────┴──────┐                                          │
│    │  Host Port  │                                          │
│    │    8087     │                                          │
│    └─────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Component | Version |
|-----------|---------|
| Java | 21 LTS (Amazon Corretto) |
| Spring Boot | 4.1.1 |
| Maven | 3.9+ |
| Docker | 24.0+ |
| Docker Compose | v2 |
| MongoDB | 7.0 |
| Base Image | amazoncorretto:21 |

---

## Project Structure

```
workshop-repo1/
├── src/
│   └── main/
│       └── java/
│           └── co/edu/escuelaing/
│               ├── HelloRestController.java
│               └── RestServiceApplication.java
├── target/
│   └── virtualization-lab-1.0.0.jar
├── Dockerfile
├── compose.yaml
├── pom.xml
├── .gitignore
├── img/
│   ├── container_images_curl.png
│   ├── dockercompose_greeting.png
│   ├── docker_compose.png
│   ├── dockerhub_push.png
│   ├── dockerimage.png
│   ├── docker_images.png
│   └── mongo_dockercompose_test.png
└── README.md
```

---

## Part 1: Spring Boot Application

### REST Endpoint

**GET** `/greeting?name={name}`

```java
@RestController
public class HelloRestController {
    @GetMapping("/greeting")
    public String greeting(
            @RequestParam(value = "name", defaultValue = "World") String name) {
        return "Hello, " + name + "!";
    }
}
```

### Application Entry Point

The application reads its port from the `PORT` environment variable (default: **6000**):

```java
@SpringBootApplication
public class RestServiceApplication {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(RestServiceApplication.class);
        application.setDefaultProperties(
                Map.of("server.port", System.getenv().getOrDefault("PORT", "6000")));
        application.run(args);
    }
}
```

### Build & Run Locally

```bash
mvn clean package
java -jar target/virtualization-lab-1.0.0.jar
```

**Test**: `http://localhost:6000/greeting?name=Karo` → `Hello, Karo!`

---

## Part 2: Docker Image

### Dockerfile

```dockerfile
FROM amazoncorretto:21
WORKDIR /app
COPY target/*.jar app.jar
ENV PORT=9000
EXPOSE 9000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build Image

```bash
docker build -t carolinacepeda/virtualization-lab:1.0 .
```

### Run Isolated Containers

```bash
# Container 1
docker run -d --name virtualization-lab-1 -e PORT=9000 -p 34000:9000 carolinacepeda/virtualization-lab:1.0

# Container 2
docker run -d --name virtualization-lab-2 -p 34001:9000 carolinacepeda/virtualization-lab:1.0

# Container 3
docker run -d --name virtualization-lab-3 -p 34002:9000 carolinacepeda/virtualization-lab:1.0
```

### Verify Isolation

| Container | Port | Test URL |
|-----------|------|----------|
| virtualization-lab-1 | 34000 | `http://localhost:34000/greeting?name=Container` |
| virtualization-lab-2 | 34001 | `http://localhost:34001/greeting?name=Container2` |
| virtualization-lab-3 | 34002 | `http://localhost:34002/greeting?name=Container3` |

Each container runs independently with its own filesystem, network stack, and process space.

---

## Part 3: Docker Compose

### compose.yaml

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: virtualization-web
    environment:
      PORT: 9000
      SPRING_DATA_MONGODB_URI: mongodb://db:27017/workshop
    ports:
      - "8087:9000"
    depends_on:
      - db

  db:
    image: mongo:7
    container_name: virtualization-db
    volumes:
      - mongodb:/data/db
      - mongodb_config:/data/configdb
    ports:
      - "27017:27017"
    command: mongod

volumes:
  mongodb:
  mongodb_config:
```

### Start Services

```bash
docker compose up -d --build
```

### Verify

```bash
docker compose ps
docker compose logs web
docker compose logs db
```

### Test Web Endpoint

```bash
curl http://localhost:8087/greeting?name=Compose
# Response: Hello, Compose!
```

### Test MongoDB Integration

```bash
# Insert document
docker exec virtualization-db mongosh workshop --eval "db.messages.insertOne({ message: 'Hello from Docker Compose' })"

# Query documents
docker exec virtualization-db mongosh workshop --eval "db.messages.find()"
```

**Output:**
```json
[
  {
    _id: ObjectId('...'),
    message: 'Hello from Docker Compose'
  }
]
```

### Data Persistence

Volumes `mongodb` and `mongodb_config` preserve data across container restarts:

```bash
docker compose down
docker compose up -d
# Data persists
docker exec virtualization-db mongosh workshop --eval "db.messages.find()"
```

### Cleanup

```bash
# Keep volumes (data preserved)
docker compose down

# Remove volumes (data deleted)
docker compose down -v
```

---

## Part 4: Docker Hub

### Login & Push

```bash
docker login

docker tag carolinacepeda/virtualization-lab:1.0 carolinacepeda/virtualization-lab:latest

docker push carolinacepeda/virtualization-lab:1.0
docker push carolinacepeda/virtualization-lab:latest
```

### Repository

**Docker Hub**: [carolinacepeda/virtualization-lab](https://hub.docker.com/r/carolinacepeda/virtualization-lab)

Tags available:
- `1.0` - Versioned release
- `latest` - Latest build

---

## Part 5: AWS EC2 Deployment



### Steps

1. **Create EC2 Instance**
   - AMI: Amazon Linux 2023
   - Type: t3.micro (Free Tier)
   - Region: us-east-1
   - Security Group: SSH (22) from my IP, HTTP (8080) from my IP

2. **Install Docker on EC2**
   ```bash
   sudo yum update -y
   sudo yum install -y docker
   sudo service docker start
   sudo usermod -a -G docker ec2-user
   ```

3. **Deploy Application**
   ```bash
   docker pull carolinacepeda/virtualization-lab:1.0
   docker run -d --name virtualization-lab --restart unless-stopped -e PORT=9000 -p 8080:9000 carolinacepeda/virtualization-lab:1.0
   ```

4. **Verify**
   - `http://<ec2-public-dns>:8080/greeting?name=AWS`

---

## Part 6: Cost Analysis


### Scenarios

| Scenario | Requests/mo | Instance Type | Hours/mo | EBS | Est. Data Transfer |
|----------|-------------|---------------|----------|-----|-------------------|
| Small | 10,000 | t3.micro | 730 | 8 GB | 1 GB |
| Medium | 100,000 | t3.small | 730 | 20 GB | 10 GB |
| Large | 1,000,000 | t3.medium | 730 | 50 GB | 100 GB |


### Analysis Questions

1. Why does EC2 have baseline cost even with few requests?
2. At what workload does fixed cost become negligible per request?
3. What triggers moving from 1 to multiple instances?
4. What production services are needed (ALB, RDS, CloudWatch, ECR, Backup)?
5. Would serverless (Lambda/Fargate) be more cost-effective for small workload?

---

## How to Run

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker Desktop / Docker Engine + Compose v2
- Docker Hub account (for push)

### Local Development

```bash
# Clone
git clone <this-repo>
cd workshop-repo1

# Build JAR
mvn clean package

# Run locally (JAR)
java -jar target/virtualization-lab-1.0.0.jar
# Test: http://localhost:6000/greeting?name=Karo
```

### Docker (Single Container)

```bash
# Build
docker build -t virtualization-lab:local .

# Run
docker run -d -p 6000:9000 -e PORT=9000 virtualization-lab:local
# Test: http://localhost:6000/greeting?name=Docker
```

### Docker Compose (Full Stack)

```bash
# Start all services
docker compose up -d --build

# Test web
curl http://localhost:8087/greeting?name=Compose

# Test MongoDB
docker exec virtualization-db mongosh workshop --eval "db.messages.find()"

# Stop (keep data)
docker compose down

# Stop (remove data)
docker compose down -v
```

### Use Published Image

```bash
docker pull carolinacepeda/virtualization-lab:1.0
docker run -d -p 8080:9000 -e PORT=9000 carolinacepeda/virtualization-lab:1.0
```

---

## Evidence

### Part 1: Local Spring Boot Execution

**Docker Image Build** - Shows the successful Maven build and JAR creation:

![Docker Image Build](img/dockerimage.png)

### Part 2: Docker Image & Isolated Containers

**Docker Images** - `docker images` output showing the built image:

![Docker Images](img/docker_images.png)

**Container Curl Tests** - Three containers responding independently on ports 34000, 34001, and 34002:

![Container Curl Tests](img/container_images_curl.png)

### Part 3: Docker Compose

**Docker Compose Up** - `docker compose up -d --build` output showing both services starting:

![Docker Compose Up](img/docker_compose.png)

**Compose Greeting** - Web endpoint responding on port 8087:

![Compose Greeting](img/dockercompose_greeting.png)

**MongoDB Test** - MongoDB insert/find verification showing data persistence:

![MongoDB Test](img/mongo_dockercompose_test.png)

### Part 4: Docker Hub

**Docker Hub Push** - Successful push of both `1.0` and `latest` tags to Docker Hub:

![Docker Hub Push](img/dockerhub_push.png)

---
