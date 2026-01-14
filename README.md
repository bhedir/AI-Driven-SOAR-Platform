# AWS Sentinel AI-Risk Remediation

A SOAR (Security Orchestration, Automation, and Response) platform for automated detection, risk assessment, and incident response using open-source tools deployed on AWS infrastructure.



## Overview

With the rapid growth of information systems and cloud infrastructures, organizations are increasingly exposed to sophisticated cyber threats. This project presents the design and implementation of a SOAR platform that integrates multiple open-source security tools to create an end-to-end automated incident response pipeline.

**Global Workflow:**

Graylog → n8n → AI Engine → Decision Engine → Response Actions

The platform automatically detects security incidents (such as SSH brute force attacks), analyzes their impact, assesses risk levels, and initiates appropriate response actions without requiring constant human intervention.

## Features

- **Centralized Log Collection**: Collect logs from Linux and Windows systems using Graylog SIEM
- **Real-time Detection**: Aggregation-based alerts for SSH brute force and other attack patterns
- **AI-Powered Risk Assessment**: FastAPI-based engine for intelligent risk classification
- **ISO/IEC 27001/27002 Mapping**: Automatic alignment with international security standards
- **Automated Response Actions**:
  - Email notifications to SOC team
  - Automatic IP blocking via Fail2Ban
  - Incident case creation in TheHive
- **Containerized Deployment**: Full Docker/Docker Compose support
- **AWS Ready**: Designed for deployment on AWS EC2 infrastructure

## Architecture
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Ubuntu/Windows │────▶│ Graylog │────▶│ n8n │
│ Log Sources │ │ SIEM │ │ Orchestration │
└─────────────────┘ └─────────────────┘ └────────┬────────┘
│
▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ TheHive │◀────│ Decision │◀────│ AI Engine │
│ Case Management │ │ Engine │ │ (FastAPI) │
└─────────────────┘ └─────────────────┘ └─────────────────┘
│
▼
┌───────────────────────┐
│ Response Actions │
│ - Email Notification │
│ - IP Blocking │
│ - Host Isolation │
└───────────────────────┘

## Components

| Component | Technology | Purpose | Port |
|-----------|------------|---------|------|
| SIEM | Graylog 5.0 | Log collection, correlation, alerting | 9000 |
| Database | MongoDB 5.0 | Graylog metadata storage | 27017 |
| Search Engine | OpenSearch 2 | Log indexing and search | 9200 |
| Orchestration | n8n | Workflow automation and SOAR logic | 5678 |
| AI Engine | FastAPI (Python) | Risk assessment and classification | 8000 |
| Case Management | TheHive 5 | Incident tracking and investigation | 9001 |

## Prerequisites

- Ubuntu Server 20.04+ (or compatible Linux distribution)
- Docker Engine 20.10+
- Docker Compose 2.0+
- Minimum 8GB RAM per component server
- AWS Account (for cloud deployment)

## Installation

### Docker Setup

Install Docker and Docker Compose on Ubuntu:

```bash
# Update system packages
sudo apt update

# Install Docker
sudo apt install -y docker.io docker-compose

# Enable and start Docker service
sudo systemctl enable docker
sudo systemctl start docker

# Verify installation
docker --version
docker-compose --version
```
### Graylog Deployment
## Graylog Deployment

Create `docker-compose-graylog.yml`:

```yaml
version: '3'
services:
  mongo:
    image: mongo:5.0
    container_name: graylog-mongo
    restart: always
    volumes:
      - mongo_data:/data/db

  opensearch:
    image: opensearchproject/opensearch:2
    container_name: graylog-opensearch
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
    container_name: graylog
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
      - "9000:9000"        # Web interface
      - "1514:1514/udp"    # Syslog UDP (Linux)
      - "12201:12201/udp"  # GELF UDP
      - "5044:5044"        # Beats (Windows)
    restart: always
    volumes:
      - graylog_data:/usr/share/graylog/data

volumes:
  mongo_data:
  opensearch_data:
  graylog_data:
```
# AWS Sentinel AI Risk Remediation

## Generate Password Hash

```bash
echo -n "your_password" | sha256sum | cut -d" " -f1
```

## Graylog Deployment

```bash
docker-compose -f docker-compose-graylog.yml up -d
```

---

## n8n Deployment

### Create `docker-compose-n8n.yml`

```yaml
version: '3'
services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
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

### Deploy n8n

```bash
docker-compose -f docker-compose-n8n.yml up -d
```

---

## TheHive Deployment

### Create `docker-compose-thehive.yml`

```yaml
version: '3'
services:
  thehive:
    image: strangebee/thehive:5
    container_name: thehive
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

### Deploy TheHive

```bash
docker-compose -f docker-compose-thehive.yml up -d
```

---

## AI Engine Deployment

### Create `app/main.py`

```python
from fastapi import FastAPI
from pydantic import BaseModel
from datetime import datetime

app = FastAPI(
    title="AI Risk Assessment Engine",
    description="SOAR AI-based risk classification service",
    version="1.0.0"
)

class Incident(BaseModel):
    title: str
    description: str

class RiskAssessment(BaseModel):
    risk_level: str
    risk_score: int
    iso_control: str
    recommendation: str
    status: str
    timestamp: str

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}

@app.post("/classify", response_model=RiskAssessment)
async def classify_incident(incident: Incident):
    message = incident.description.lower()
    risk_score = 1

    if "ssh" in message:
        risk_score += 2
    if "brute force" in message or "brute-force" in message:
        risk_score += 2
    if "count()=5" in message or "count()=10" in message:
        risk_score += 1
    if "count()=15" in message or "count()=20" in message:
        risk_score += 2
    if "authentication failure" in message or "failed password" in message:
        risk_score += 1
    if "sudo" in message or "root" in message:
        risk_score += 1

    if risk_score >= 5:
        return RiskAssessment(
            risk_level="HIGH",
            risk_score=risk_score,
            iso_control="ISO/IEC 27001 A.5.24 Incident response planning",
            recommendation="Block source IP, isolate host, investigate authentication logs",
            status="Critical",
            timestamp=datetime.utcnow().isoformat()
        )
    elif risk_score >= 3:
        return RiskAssessment(
            risk_level="MEDIUM",
            risk_score=risk_score,
            iso_control="ISO/IEC 27001 A.5.25 Assessment and decision on information security events",
            recommendation="Monitor closely, consider temporary blocking, review logs",
            status="Warning",
            timestamp=datetime.utcnow().isoformat()
        )
    else:
        return RiskAssessment(
            risk_level="LOW",
            risk_score=risk_score,
            iso_control="ISO/IEC 27002 A.8.15 Logging and monitoring",
            recommendation="Continue monitoring and review logs",
            status="Informational",
            timestamp=datetime.utcnow().isoformat()
        )
```

### Create `requirements.txt`

```text
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.2
```

### Create `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Create `docker-compose-ai.yml`

```yaml
version: '3'
services:
  ai-engine:
    build: .
    container_name: ai-risk-engine
    ports:
      - "8000:8000"
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Deploy AI Engine

```bash
docker-compose -f docker-compose-ai.yml up -d --build
```

---

## AWS Infrastructure Setup

### EC2 Instances

| Instance         | Type      | Purpose                             |
| ---------------- | --------- | ----------------------------------- |
| graylog-server   | t3.medium | Graylog SIEM + MongoDB + OpenSearch |
| n8n-server       | t3.small  | n8n orchestration engine            |
| ai-engine-server | t3.small  | FastAPI risk assessment             |
| thehive-server   | t3.small  | TheHive case management             |
| ubuntu-client    | t3.micro  | Linux log source                    |
| windows-client   | t3.small  | Windows log source                  |

### Security Groups

| Port | Protocol | Source   | Purpose            |
| ---- | -------- | -------- | ------------------ |
| 22   | TCP      | Your IP  | SSH administration |
| 1514 | UDP      | VPC CIDR | Syslog from Linux  |
| 5044 | TCP      | VPC CIDR | Beats from Windows |
| 9000 | TCP      | VPC CIDR | Graylog UI         |
| 5678 | TCP      | VPC CIDR | n8n UI             |
| 8000 | TCP      | VPC CIDR | AI Engine API      |
| 9001 | TCP      | VPC CIDR | TheHive UI         |
| 3389 | TCP      | Your IP  | RDP                |

---

## Graylog Configuration

### Linux Syslog Input

* Type: Syslog UDP
* Port: 1514
* Purpose: Ubuntu authentication and system logs

### Windows Log Input

* Type: Beats or GELF
* Port: 5044 or 12201
* Purpose: Windows Security Event Logs

### SSH Brute Force Detection

* Title: **Linux SSH Brute Force**
* Condition: Aggregation
* Search Query: `application_name:sshd AND message:"Failed password"`
* Group By: `source`
* Threshold: `count > 5 within 5 minutes`

Notification:

* Type: HTTP
* URL: `http://<N8N_IP>:5678/webhook/graylog-alert`

---

## n8n Workflow

### Function Node (Normalization)

```javascript
const event = $input.first().json;

return {
  incident_type: event.event_definition_title || "Unknown",
  src_ip: event.event?.fields?.source || "Unknown",
  host: event.event?.fields?.host || "Unknown",
  timestamp: event.event?.timestamp || new Date().toISOString(),
  graylog_priority: event.event?.priority || 0,
  message: event.event?.message || ""
};
```

### AI Engine Request

```json
{
  "title": "{{ $json.incident_type }}",
  "description": "{{ $json.message }}"
}
```

---

## Testing

### Simulate SSH Brute Force

```bash
for i in {1..10}; do
  ssh invalid_user@<TARGET_IP>
done
```

### API Test

```bash
curl -X POST "http://<AI_ENGINE_IP>:8000/classify" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Linux SSH Brute Force",
    "description": "Linux SSH Brute Force: 192.168.1.100 - count()=10"
  }'
```

---

## ISO/IEC Compliance Mapping

| Risk Level | ISO Control | Description                       |
| ---------- | ----------- | --------------------------------- |
| LOW        | A.8.15      | Logging and monitoring            |
| MEDIUM     | A.5.25      | Assessment and decision on events |
| HIGH       | A.5.24      | Incident response planning        |

---

## Project Structure

```text
aws-sentinel-ai-risk-remediation/
├── docker-compose-graylog.yml
├── docker-compose-n8n.yml
├── docker-compose-thehive.yml
├── docker-compose-ai.yml
├── ai-engine/
│   ├── app/
│   │   └── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── n8n-workflows/
│   └── soar-workflow.json
├── graylog-config/
│   └── alert-definitions/
│       └── ssh-brute-force.json
├── docs/
│   └── architecture-diagrams/
└── README.md
```

---

## License

MIT License

## Author

Hedir Bel Arbia
January 2026


