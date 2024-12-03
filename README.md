# Deploying and Monitoring Full-Stack Applications with Docker, Nginx, and Grafana in the Cloud

## Introduction

In this guide, we will explore how to deploy a full-stack application using Docker Compose while integrating monitoring and logging tools to track the application's health and gain meaningful insights from the logs. This comprehensive setup includes:

- **Prometheus**: A tool designed for collecting and storing time-series data (e.g., metrics like disk usage or CPU performance).
- **Grafana**: A visualization and monitoring solution that allows you to create stunning dashboards and gain insights into metrics and logs.
- **Nginx Proxy Manager**: A user-friendly reverse proxy solution to simplify domain and SSL management.

By the end, you will have a scalable, secure, and monitored application deployed in the cloud.

---

## Foundational Requirements

Before we begin, ensure you have the following:

1. Basic understanding of Linux commands.
2. Knowledge of setting up an EC2 instance and configuring security groups.
3. Familiarity with Prometheus and Grafana.
4. Understanding of essential Docker commands and installing Docker on an EC2 instance.
5. A registered domain name for deployment.
6. Development tools such as Gitbash or Visual Studio Code (VSCode).

---
## Project Overview

### Deployment Goals

1. **Full-Stack Application**:
   - React frontend
   - FastAPI backend
   - PostgreSQL database
2. **Monitoring Stack**:
   - Prometheus for metrics scraping
   - Grafana for dashboard visualization
   - Loki and Promtail for centralized logging
   - cAdvisor for container monitoring
3. **Reverse Proxy**:
   - Nginx Proxy Manager for routing, SSL, and domain management

---

## Step 1: Setting Up the Environment

### 1.1 Launching an EC2 Instance
1. Create an Ubuntu EC2 instance with the following specs:
   - Name: `FullStackApp`
   - Instance Type: `t2.small`
   - Security groups: Ensure ports 80, 443, 3000, 8000, 5173, and 8081 are open.
2. Connect to the instance via SSH using Gitbash or VSCode:
   ```bash
   ssh -i <your-key.pem> ubuntu@<your-ec2-public-ip>
   ```

### 1.2 Install Docker and Docker Compose
```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install docker.io -y
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

---

## Step 2: Containerizing the Full-Stack Application

### 2.1 Backend Dockerfile
Create a `Dockerfile` in the `backend` directory with the following content:
```dockerfile
FROM python:3.10
WORKDIR /app
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
RUN curl -sSL https://install.python-poetry.org | python3 -
ENV PATH="/root/.local/bin:$PATH"
COPY . /app
RUN poetry install
RUN chmod +x ./prestart.sh
EXPOSE 8000
CMD ["sh", "-c", "poetry run bash ./prestart.sh && poetry run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload"]
```

### 2.2 Frontend Dockerfile
Create a `Dockerfile` in the `frontend` directory with the following content:
```dockerfile
FROM node:latest
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]
```

### 2.3 Docker Compose File
Create a `docker-compose.yml` file in the root directory:
```yaml
version: '3.8'
services:
  frontend:
    build:
      context: ./frontend
    env_file:
      - frontend/.env
    depends_on:
      - backend
    ports:
      - "5173:5173"
    networks:
      - frontend-network

  backend:
    build:
      context: ./backend
    env_file:
      - backend/.env
    networks:
      - frontend-network
      - backend-network
    depends_on:
      - db
    secrets:
      - postgres_password
    ports:
      - "8000:8000"

  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      POSTGRES_USER: app
      POSTGRES_DB: app
    secrets:
      - postgres_password
    networks:
      - backend-network
    volumes:
      - postgres_data:/var/lib/postgresql/data

networks:
  frontend-network:
  backend-network:

volumes:
  postgres_data:

secrets:
   postgres_password:
     file: ./POSTGRES_PASSWORD.txt
```

---

## Step 3: Setting Up Monitoring

### 3.1 Prometheus Configuration
Create a `prometheus.yml` file in a `monitoring` directory:
```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

### 3.2 Loki Configuration
Create a `loki-config.yml` file:
```yaml
auth_enabled: false
server:
  http_listen_port: 3100
common:
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v12
```

### 3.3 Docker Compose Monitoring File
Create a `compose.monitoring.yml` file:
```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana-oss
    ports:
      - "3000:3000"
    volumes:
      - grafana:/var/lib/grafana

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    ports:
      - "8081:8080"

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

volumes:
  grafana:
```

---

## Step 4: Deploying the Application

### 4.1 Build and Run Containers
Run the following commands:
```bash
docker-compose -f docker-compose.yml -f compose.monitoring.yml up -d
```

### 4.2 Verify the Setup
- Access the frontend at `http://<your-ec2-public-ip>:5173`.
- Access the backend at `http://<your-ec2-public-ip>:8000`.
- Access Grafana at `http://<your-ec2-public-ip>:3000`.
- Verify Prometheus and cAdvisor at their respective ports.

---

## Step 5: Configuring Nginx Proxy Manager

### 5.1 Deploy Nginx Proxy Manager
Add the following service to your `docker-compose.yml`:
```yaml
  nginx:
    image: 'jc21/nginx-proxy-manager:latest'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - data:/data
      - letsencrypt:/etc/letsencrypt
    restart: always
```

### 5.2 Secure the Application with a Domain
1. Login to Nginx Proxy Manager.
2. Add your domain and configure SSL.

---

## Conclusion

By following this guide, you successfully deployed a monitored, secure, and scalable full-stack application in the cloud. The integration of Prometheus, Grafana, and Nginx Proxy Manager ensures real-time insights and easy management of your application. This setup can be a template for deploying enterprise-grade cloud applications.
