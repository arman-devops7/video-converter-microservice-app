# DevOps Project: Video Converter

## Introduction

This document provides a comprehensive guide for deploying a Python-based microservice application on AWS Elastic Kubernetes Service (EKS). The project involves converting MP4 videos to MP3 format using a microservices architecture. The application comprises four primary microservices:

1. **Auth Service**: Handles user authentication.
2. **Converter Service**: Converts video files (MP4) to audio files (MP3).
3. **Database Service**: Utilizes PostgreSQL and MongoDB for data storage.
4. **Notification Service**: Sends notifications and two-factor authentication (2FA) emails.

This guide is structured to demonstrate the project’s workflow, architecture, and deployment strategy, showcasing your DevOps skills effectively.

---

## Architecture

<p align="center">
  <img src="./Project documentation/ProjectArchitecture.png" width="600" title="Architecture" alt="Architecture">
</p>

The architecture consists of the following components:
- **EKS Cluster**: Hosts the microservices.
- **RabbitMQ**: Manages message queues for file conversion tasks.
- **PostgreSQL**: Stores authentication-related data.
- **MongoDB**: Stores metadata about uploaded videos and converted audio files.

---

## Prerequisites

Before you begin, ensure the following tools and accounts are set up:

1. **AWS Account**: [Set up an AWS account](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html).
2. **AWS CLI**: [Install the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
3. **kubectl**: [Install kubectl](https://kubernetes.io/docs/tasks/tools/).
4. **Helm**: [Install Helm](https://helm.sh/docs/intro/install/).
5. **Python**: [Install Python](https://www.python.org/downloads/).
6. **Databases**: Ensure PostgreSQL and MongoDB are installed or available for connection.

---

## Deployment Steps

### High-Level Workflow

1. **Cluster Setup**:
   - Create an EKS cluster and associated IAM roles.
   - Configure networking and node groups.
2. **Database Setup**:
   - Deploy MongoDB and PostgreSQL using Helm charts.
3. **Message Queue Setup**:
   - Deploy RabbitMQ and configure necessary queues (`mp3` and `video`).
4. **Microservice Deployment**:
   - Deploy each microservice using Kubernetes manifests.
5. **Application Validation**:
   - Test the application by interacting with its APIs.
6. **Clean-Up**:
   - Tear down the infrastructure after testing.

---

### Low-Level Details

#### Cluster Creation

1. **Log in to AWS Console**:
   - Use your AWS credentials to access the Management Console.

2. **Create eksCluster IAM Role**:
   - Follow [this guide](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html).
   - Ensure the `AmazonEKS_CNI_Policy` is explicitly attached.

<p align="center">
  <img src="./Project documentation/ekscluster_role.png" width="600" title="ekscluster_role" alt="ekscluster_role">
</p>

3. **Create Node IAM Role**:
   - Refer to [this documentation](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html).
   - Attach the following policies:
     - `AmazonEKS_CNI_Policy`
     - `AmazonEBSCSIDriverPolicy`
     - `AmazonEC2ContainerRegistryReadOnly`

<p align="center">
  <img src="./Project documentation/node_iam.png" width="600" title="Node_IAM" alt="Node_IAM">
</p>

4. **Create EKS Cluster**:
   - Navigate to the Amazon EKS dashboard.
   - Create a cluster with the appropriate networking settings and select the `eksCluster` IAM role.

5. **Enable Add-Ons**:
   - Enable the `EBS CSI` add-on for persistent volume support.

<p align="center">
  <img src="./Project documentation/ebs_addon.png" width="600" title="ebs_addon" alt="ebs_addon">
</p>

6. **Add Node Groups**:
   - Configure node groups with instance types (e.g., `t3.medium`) and specify the desired node count.

<p align="center">
  <img src="./Project documentation/inbound_rules_sg.png" width="600" title="Inbound_rules_sg" alt="Inbound_rules_sg">
</p>

---

#### Database Deployment

- **MongoDB**:
  1. Set username and password in `values.yaml`.
  2. Navigate to the MongoDB Helm chart folder and install MongoDB:
     ```bash
     helm install mongo .
     ```
  3. Connect to MongoDB:
     ```bash
     mongosh mongodb://<username>:<pwd>@<nodeip>:30005/mp3s?authSource=admin
     ```

- **PostgreSQL**:
  1. Configure `values.yaml` for username and password.
  2. Deploy PostgreSQL:
     ```bash
     helm install postgres .
     ```
  3. Initialize the database with queries in `init.sql`:
     ```bash
     psql 'postgres://<username>:<pwd>@<nodeip>:30003/authdb'
     ```

---

#### RabbitMQ Deployment

1. Deploy RabbitMQ:
   ```bash
   helm install rabbitmq .
   ```
2. Create queues named `mp3` and `video` via the RabbitMQ dashboard at `<nodeIp>:30004`.

---

#### Microservice Deployment

1. **Auth Service**:
   ```bash
   cd auth-service/manifest
   kubectl apply -f .
   ```
2. **Gateway Service**:
   ```bash
   cd gateway-service/manifest
   kubectl apply -f .
   ```
3. **Converter Service**:
   ```bash
   cd converter-service/manifest
   kubectl apply -f .
   ```
4. **Notification Service**:
   ```bash
   cd notification-service/manifest
   kubectl apply -f .
   ```

---

### Application Validation

Run the following command to check the status of all components:
```bash
kubectl get all
```

---

## Notification Configuration

1. Enable 2-Step Verification in your Gmail account.
2. Generate an application-specific password.
3. Update `notification-service/manifest/secret.yaml` with your email and the generated password.

---

## API Definitions

- **Login**:
  ```bash
  curl -X POST http://<nodeIP>:30002/login -u <email>:<password>
  ```
- **Upload**:
  ```bash
  curl -X POST -F 'file=@./video.mp4' -H 'Authorization: Bearer <JWT Token>' http://<nodeIP>:30002/upload
  ```
- **Download**:
  ```bash
  curl --output video.mp3 -X GET -H 'Authorization: Bearer <JWT Token>' "http://<nodeIP>:30002/download?fid=<Generated file identifier>"
  ```

---

## Cleaning Up Infrastructure

1. Delete the node group associated with your EKS cluster.
2. Delete the EKS cluster.
3. Clean up any additional resources, such as security groups and volumes.

