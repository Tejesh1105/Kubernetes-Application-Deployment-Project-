Production Deployment of Multi-Tier Application on Kubernetes (VProfile Stack)
1. Project Overview
This project focuses on the robust, production-ready deployment of a pre-containerized multi-tier Java web application (VProfile stack) onto an Amazon Web Services (AWS)-hosted Kubernetes cluster. The core objective is to leverage the full power of Kubernetes for orchestration, ensuring High Availability (HA), Fault Tolerance, and Scalability across all application services.

The entire deployment is defined through Kubernetes manifest files, making the environment platform-independent, portable, and fully agile across different environments (Dev, QA, Production).

2. Key Architectural Requirements
The deployment strategy is engineered to meet critical non-functional requirements essential for a production environment:

Requirement

Kubernetes Solution

Benefit

High Availability

Deployment objects with replicas > 1 (for stateless components).

Containers are automatically rescheduled if a node fails.

Fault Tolerance & Auto-Healing

Deployment controllers manage the desired state.

Unhealthy containers are automatically replaced or restarted.

Scalability

Easy adjustment of replicas count via Deployment manifest.

Horizontal Pod Autoscaling (HPA) can be easily implemented.

Persistence (Stateful)

Persistent Volume Claims (PVC) leveraging AWS EBS volumes.

Guarantees data survival independent of the Pod lifecycle.

Secure Configuration

Kubernetes Secret objects.

Safely stores sensitive information like database credentials.

3. Technology Stack
Category

Component

Purpose

Orchestration

Kubernetes

Container management, scaling, and self-healing.

Infrastructure

AWS (EKS/kops)

Hosting environment and Elastic Block Storage (EBS).

Application

VProfile Java Stack

Multi-tier application (Web, Database, Caching, Messaging).

Backend Services

MySQL, Memcached, RabbitMQ

Core application data and messaging components.

Networking

ClusterIP Service, Nginx Ingress Controller, AWS ALB

Internal load balancing and external traffic routing.

4. Kubernetes Architecture Breakdown
The application stack is broken down into several Kubernetes objects to ensure proper isolation, communication, and state management.

A. Core Services (Deployments & Services)
Each tier of the application stack (Tomcat, MySQL, Memcache, RabbitMQ) is managed by a Kubernetes Deployment object (ensuring replicas and healing) and exposed internally via a Service of type ClusterIP.

Tomcat Deployment: Hosts the stateless VProfile web application.

Backend Deployments (MySQL, Memcache, RabbitMQ): Run their respective containers.

Service (ClusterIP): Provides a stable, internal DNS name and load balancing endpoint for the Tomcat pod to connect to any backend pod (e.g., vprodb:3306).

B. Stateful Management (MySQL Persistence)
The MySQL database is a Stateful component. To prevent data loss upon pod failure or deletion, we use:

PersistentVolumeClaim (PVC): Requests a specific amount of storage (e.g., 10GB).

StorageClass (AWS EBS): Dynamically provisions the required EBS volume on AWS and binds it to the PVC.

Volume Mount: The PVC is mounted inside the MySQL pod at the default storage path (/var/lib/mysql).

C. Security and Configuration
Secrets: A Kubernetes Secret object (type Opaque) is used to store sensitive passwords (e.g., MySQL root password, RabbitMQ user password). These secrets are injected into the respective Pods as environment variables during runtime, ensuring they are not exposed in plaintext within the manifest files.

D. External Access and Routing
External access to the web application is managed through two components:

Ingress Controller (Nginx): Manages external traffic routing.

Ingress Resource: Defines the routing rules.

The Ingress Controller automatically provisions an AWS Application Load Balancer (ALB).

An Ingress Rule specifies that traffic matching a specific host (e.g., vprofile.mywebsite.com) should be routed to the internal Tomcat Service.

5. Critical Implementation Details
This deployment incorporates specific solutions for common production challenges:

Node Affinity for EBS Volumes
Since AWS EBS volumes are zone-specific, the database pod must always run on a worker node located within the same availability zone as the provisioned EBS volume.

Solution: Worker nodes are explicitly labeled with their zone names (e.g., failure-domain.beta.kubernetes.io/zone: us-east-1a). The database deployment manifest uses a Node Selector to ensure the pod is scheduled only on a node matching the necessary zone label.

EBS Volume Cleanup (lost+found)
When a new EBS volume is formatted, the Linux filesystem often creates a non-empty lost+found directory. This can cause the MySQL service to fail on startup when mounting to /var/lib/mysql.

Solution: The MySQL Deployment utilizes an initContainer. This ephemeral container runs before the main MySQL container. Its sole job is to mount the PVC and execute a command (rm -rf /var/lib/mysql/lost+found) to ensure the volume is truly empty before the database service initializes.

Application Initialization Delay
To ensure frontend components do not start before essential backend services (like the database) are fully ready and accepting connections:


