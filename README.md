# eDiscovery - WhatsApp Legal Hold Platform

eDiscovery is a compliance platform designed to capture, store, and retrieve WhatsApp instant messenger conversations for businesses. It helps organizations comply with regulations like GDPR and HIPAA by providing secure archival and efficient retrieval of communications.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
  - [Backend](#backend)
  - [Frontend](#frontend)
- [Prerequisites](#prerequisites)
- [Local Development](#local-development)
- [Building and Running](#building-and-running)
  - [Using Docker](#using-docker)
  - [Using Docker Compose](#using-docker-compose)
  - [Running Locally](#running-locally)
- [Deployment](#deployment)
  - [AWS Deployment](#aws-deployment)
  - [GCP Deployment](#gcp-deployment)
- [Workflow](#workflow)
- [API Endpoints](#api-endpoints)
- [Infrastructure](#infrastructure)

## Overview

The eDiscovery platform provides:

- **Automatic Message Capture**: Seamless integration with WhatsApp using the `whatsmeow` library
- **Secure Data Storage**: SQLite database for storing user credentials and WhatsApp device information
- **Efficient Retrieval**: RESTful API for accessing chats and messages
- **QR Code Authentication**: Easy device pairing via WhatsApp Web protocol
- **User Authentication**: JWT-based authentication with HTTP-only cookies
- **Kubernetes Deployment**: Production-ready deployment on AWS EKS or GCP GKE

## Architecture

### Backend

The backend is written in **Go 1.23** and provides a RESTful API server.

**Key Components:**

- **Main Server** (`src/main.go`): HTTP server using Gorilla Mux router, runs on port 8080
- **Database Layer** (`src/database/`): SQLite database management for user and device data
- **Handlers** (`src/handlers/`):
  - `signup.go` - User registration
  - `login.go` - JWT authentication
  - `users.go` - WhatsApp device management
  - `chats.go` - Chat list retrieval
  - `messages.go` - Message retrieval with pagination
  - `qrcode.go` - WhatsApp QR code generation
- **WhatsApp Integration** (`src/meow/`): WhatsApp client management using `whatsmeow` library
- **Models** (`src/models/`): Data structures for API responses

**Technology Stack:**

- Go 1.23
- Gorilla Mux (HTTP routing)
- SQLite3 (database)
- JWT for authentication (`golang-jwt/jwt`)
- WhatsApp Web protocol (`go.mau.fi/whatsmeow`)

### Frontend

The frontend is a **static web application** located in the `static/` directory.

**Pages:**

- `index.html` - Landing page
- `signup.html` - User registration
- `login.html` - User authentication
- `users.html` - WhatsApp device list
- `chats.html` - Chat list for a device
- `messages.html` - Message view with pagination
- `onboard.html` - QR code device pairing
- `demo.html` - Demo/presentation page

**Technology Stack:**

- HTML/CSS/JavaScript
- Client-side routing and API calls

## Prerequisites

- **Go 1.23+** - For local development
- **Docker** - For containerization
- **Docker Compose** - For local stack with ELK
- **Terraform 1.3+** - For infrastructure deployment
- **kubectl** - For Kubernetes interaction
- **Helm 3+** - For Kubernetes package management
- **AWS CLI** - For AWS deployment
- **gcloud CLI** - For GCP deployment

## Local Development

### Clone the Repository

```bash
git clone https://github.com/dkovacevic/ediscovery.git
cd ediscovery
```

### Install Dependencies

```bash
go mod download
```

### Run the Application

```bash
go run src/main.go
```

The server will start at `http://localhost:8080`.

## Building and Running

### Using Docker

Build the Docker image:

```bash
docker buildx build --platform linux/amd64 -t dejankovacevic/ediscovery:0.2.4 .
```

Run the container:

```bash
docker run -p 8080:8080 -v $(pwd)/data:/opt/ediscovery/data dejankovacevic/ediscovery:0.2.4
```

### Using Docker Compose

The `docker-compose.yml` includes the application along with ELK stack (Elasticsearch, Kibana, Filebeat) for log aggregation:

```bash
docker-compose up -d
```

Services:
- **eDiscovery API**: http://localhost:8080
- **Elasticsearch**: http://localhost:9200
- **Kibana**: http://localhost:5601

### Running Locally

```bash
# Create data directory for SQLite database
mkdir -p data

# Run the application
go run src/main.go
```

## Deployment

### AWS Deployment

The project includes Terraform configuration for deploying to AWS EKS (Elastic Kubernetes Service).

**Infrastructure Components:**

1. **EKS Cluster** - Kubernetes cluster on AWS
2. **VPC & Networking** - Private and public subnets with NAT gateway
3. **NGINX Ingress Controller** - Load balancer and TLS termination
4. **cert-manager** - Let's Encrypt SSL certificate management
5. **Persistent Storage** - EBS volumes for SQLite database

**Deployment Steps:**

```bash
cd terraform

# Initialize Terraform
terraform init

# Review the deployment plan
terraform plan

# Deploy infrastructure sequentially
./bootup
```

The `bootup` script executes:

```bash
terraform apply -target=module.aws_eks -auto-approve
terraform apply -target=module.ingress_helm -auto-approve
terraform apply -target=module.lh_kubernetes -auto-approve

kubectl get svc -n ingress-nginx
kubectl logs -n cert-manager deploy/cert-manager
```

**Configuration:**

Edit `terraform/variables.tf` to customize:

```hcl
variable "region" {
  default = "us-east-1"  # AWS region
}

variable "cluster_name" {
  default = "barbara"  # EKS cluster name
}

variable "app" {
  default = "ediscovery"  # Application name
}
```

**Teardown:**

```bash
cd terraform
./teardown
```

This destroys resources in reverse order:

```bash
terraform destroy -target=module.lh_kubernetes -auto-approve
terraform destroy -target=module.ingress_helm -auto-approve
terraform destroy -target=module.aws_eks -auto-approve
```

**CI/CD Pipeline:**

GitHub Actions automatically builds and pushes images to Amazon ECR on push to `main`:

- Workflow: `.github/workflows/build.yaml`
- ECR Repository: `536697232357.dkr.ecr.us-east-1.amazonaws.com/ediscovery:latest`

### GCP Deployment

The project also supports GCP deployment via GitHub Actions.

**CI/CD Pipeline:**

- Workflow: `.github/workflows/build-gcr.yaml`
- GCR Repository: `gcr.io/bustling-syntax-439308-h6/ediscovery:latest`

The workflow:
1. Authenticates to GCP using service account
2. Builds Docker image
3. Pushes to Google Container Registry

**Manual Deployment to GKE:**

```bash
# Authenticate to GCP
gcloud auth login

# Configure kubectl
gcloud container clusters get-credentials <cluster-name> --region <region>

# Apply Kubernetes manifests
kubectl apply -f kube/
```

## Workflow

### 1. User Registration & Authentication

```
User → /api/signup → Create Account → /api/login → Receive JWT Token
```

### 2. WhatsApp Device Pairing

```
User → /api/code → Generate QR Code → Scan with WhatsApp → Device Paired
```

### 3. Data Capture & Retrieval

```
WhatsApp Messages → whatsmeow Client → SQLite Database → API → Frontend
```

**API Flow:**

1. **Get Devices**: `GET /api/users` - List paired WhatsApp devices
2. **Get Chats**: `GET /api/{lhid}/chats` - List chats for a device
3. **Get Messages**: `GET /api/{lhid}/chats/{chatid}/messages` - Retrieve messages

### 4. Data Storage

- **SQLite Database**: Stored in `data/device.db`
- **Persistent Volume**: Mounted at `/opt/ediscovery/data` in Kubernetes
- **WhatsApp Sessions**: Stored for maintaining device connections

## API Endpoints

Full API documentation is available in `openAPI/openapi.yaml`.

### Authentication

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/signup` | Create new user account | No |
| POST | `/api/login` | Authenticate and receive JWT | No |

### WhatsApp Devices

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/users` | Get all paired WhatsApp devices | Yes |
| GET | `/api/code` | Generate QR code for pairing | Yes |

### Chats & Messages

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/{lhid}/chats` | Get all chats for a device | Yes |
| GET | `/api/{lhid}/chats/{chatid}/messages` | Get messages from a chat (paginated) | Yes |

**Authentication:** JWT token via HTTP-only cookie (24-hour expiry)

## Infrastructure

### Kubernetes Resources

- **Deployment**: 1 replica, persistent volume mount for database
- **Service**: NodePort on port 80 → container port 8080
- **Ingress**: NGINX with Let's Encrypt TLS
- **PersistentVolumeClaim**: 10Gi for SQLite database
- **cert-manager**: Automatic SSL certificate management

### Terraform Modules

- **`modules/eks`**: EKS cluster, VPC, subnets, node groups, EBS CSI driver
- **`modules/helm`**: NGINX Ingress Controller, cert-manager
- **`modules/kubernetes`**: Deployment, service, ingress, persistent volumes

### Monitoring

The `docker-compose.yml` includes ELK stack for local log aggregation:

- **Elasticsearch**: Log storage and search
- **Kibana**: Log visualization
- **Filebeat**: Log shipping from containers

---

## Additional Resources

- **OpenAPI Specification**: `openAPI/openapi.yaml`
- **Kubernetes Manifests**: `kube/` directory
- **Investor Deck**: `Deck.md`

## License

[License information not specified]

## Support

For issues and questions, please open an issue on GitHub.