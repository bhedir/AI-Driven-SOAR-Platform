# AWS Sentinel AI‑Risk Remediation

A **SOAR (Security Orchestration, Automation, and Response)** platform designed for automated threat detection, AI‑based risk assessment, and incident response using open‑source security tools deployed on **AWS infrastructure**.

---

## Overview

With the rapid expansion of cloud infrastructures and distributed information systems, organizations are increasingly exposed to advanced cyber threats. This project presents the **design and implementation of an end‑to‑end SOAR platform** that integrates multiple open‑source technologies to automate detection, analysis, decision‑making, and response.

### Global Workflow

```
Log Sources (Linux / Windows)
        │
        ▼
     Graylog (SIEM)
        │
        ▼
     n8n (SOAR Orchestration)
        │
        ▼
  AI Risk Engine (FastAPI)
        │
        ▼
   Decision Engine
        │
        ▼
Response Actions (Email, IP Block, TheHive)
```

The platform automatically detects security incidents (e.g., SSH brute‑force attacks), evaluates their impact, assigns a risk level, and triggers appropriate response actions **without continuous human intervention**.

---

## Key Features

* **Centralized Log Collection** using Graylog SIEM for Linux and Windows systems
* **Real‑Time Threat Detection** via aggregation‑based alerts
* **AI‑Powered Risk Assessment** using a FastAPI microservice
* **ISO/IEC 27001 & 27002 Alignment** for governance and compliance
* **Automated Incident Response**:

  * SOC email notifications
  * Automatic IP blocking (Fail2Ban)
  * Case creation in TheHive
* **Fully Containerized Architecture** using Docker & Docker Compose
* **AWS‑Ready Design** optimized for EC2 deployment

---

## Architecture Overview

```
┌──────────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Linux / Windows Logs │ ──▶ │ Graylog SIEM     │ ──▶ │ n8n Orchestration │
└──────────────────────┘     └──────────────────┘     └─────────┬────────┘
                                                                  │
                                                                  ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ TheHive          │ ◀── │ Decision Engine  │ ◀── │ AI Engine        │
│ Case Management  │     │                  │     │ (FastAPI)        │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                                  │
                                                                  ▼
┌──────────────────────────────────────────┐
│ Automated Response Actions                │
│ • Email Notification                     │
│ • IP Blocking (Fail2Ban)                 │
│ • Host Isolation / Containment           │
└──────────────────────────────────────────┘
```

---

## Core Components

| Component       | Technology       | Purpose                              | Port  |
| --------------- | ---------------- | ------------------------------------ | ----- |
| SIEM            | Graylog 5        | Log ingestion, correlation, alerting | 9000  |
| Database        | MongoDB 5        | Graylog metadata storage             | 27017 |
| Search Engine   | OpenSearch 2     | Log indexing & search                | 9200  |
| Orchestration   | n8n              | SOAR workflow automation             | 5678  |
| AI Engine       | FastAPI (Python) | Risk classification                  | 8000  |
| Case Management | TheHive 5        | Incident tracking                    | 9001  |

---

## Prerequisites

* Ubuntu Server 20.04+ (or compatible Linux distribution)
* Docker Engine ≥ 20.10
* Docker Compose ≥ 2.0
* Minimum **8 GB RAM per server** (recommended)
* AWS account with EC2 access

---

## Installation

### Docker Installation (Ubuntu)

```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker

docker --version
docker-compose --version
```

---

## Graylog Deployment

### Generate Admin Password Hash

```bash
echo -n "your_password" | sha256sum | cut -d" " -f1
```

### Create `docker-compose-graylog.yml`

```yaml
version: '3'
services:
  mongo:
    image: mongo:5.0
    restart: always
    volumes:
      - mongo_data:/data/db

  opensearch:
    image: opensearchproject/opensearch:2
    environment:
      - discovery.type=single-node
      - plugins.security.disabled=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
    ulimits:
      memlock:
        soft: -1
        hard: -1
    restart: always
    volumes:
      - opensearch_data:/usr/share/opensearch/data

  graylog:
    image: graylog/graylog:5.0
    depends_on:
      - mongo
      - opensearch
    environment:
      - GRAYLOG_HTTP_EXTERNAL_URI=http://<GRAYLOG_IP>:9000/
      - GRAYLOG_PASSWORD_SECRET=somepasswordpepper16chars
      - GRAYLOG_ROOT_PASSWORD_SHA2=<SHA256_PASSWORD>
      - GRAYLOG_ELASTICSEARCH_HOSTS=http://opensearch:9200
      - GRAYLOG_MONGODB_URI=mongodb://mongo:27017/graylog
    ports:
      - "9000:9000"
      - "1514:1514/udp"
      - "12201:12201/udp"
      - "5044:5044"
    restart: always
    volumes:
      - graylog_data:/usr/share/graylog/data

volumes:
  mongo_data:
  opensearch_data:
  graylog_data:
```

### Deploy Graylog

```bash
docker-compose -f docker-compose-graylog.yml up -d
```

---

## n8n Deployment

```yaml
version: '3'
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=<SECURE_PASSWORD>
      - N8N_HOST=<N8N_IP>
      - WEBHOOK_URL=http://<N8N_IP>:5678/
      - GENERIC_TIMEZONE=Africa/Tunis
    volumes:
      - n8n_data:/home/node/.n8n
    restart: always

volumes:
  n8n_data:
```

```bash
docker-compose -f docker-compose-n8n.yml up -d
```

---

## TheHive Deployment

```yaml
version: '3'
services:
  thehive:
    image: strangebee/thehive:5
    ports:
      - "9001:9000"
    environment:
      - THEHIVE_DB_PROVIDER=local
    volumes:
      - thehive_data:/data
    restart: always

volumes:
  thehive_data:
```

```bash
docker-compose -f docker-compose-thehive.yml up -d
```

---

## ISO/IEC Compliance Mapping

| Risk Level | ISO Control | Description                   |
| ---------- | ----------- | ----------------------------- |
| LOW        | A.8.15      | Logging and monitoring        |
| MEDIUM     | A.5.25      | Event assessment and decision |
| HIGH       | A.5.24      | Incident response planning    |

---

## Author

**Hedir Bel Arbia**
January 2026
