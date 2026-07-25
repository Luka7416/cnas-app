# CNAS Cloud Native Application Deployment

## Overview

This project demonstrates a production-style CI/CD pipeline for deploying a Dockerized PHP CRUD application onto a Kubernetes cluster hosted on AWS EC2.

The pipeline automatically:

- Builds Docker images
- Performs security scanning
- Pushes images to Docker Hub
- Deploys to Kubernetes
- Supports automated rollbacks

---

## Architecture

GitHub
↓
Jenkins
↓
Docker Build
↓
Trivy Security Scan
↓
Docker Hub
↓
Kubernetes Deployment

---

## Technologies

- AWS EC2
- Ubuntu 24.04
- Docker
- Kubernetes
- Jenkins
- GitHub
- Docker Hub

---

## Team Members

| Member | Responsibility |
|---------|----------------|
| Member 1 | Infrastructure |
| Member 2 | Kubernetes |
| Member 3 | Security & Monitoring |
| You | Secure CI/CD Pipeline |

---

## Repository Structure

```
app/
docker/
kubernetes/
jenkins/
scripts/
docs/
```
