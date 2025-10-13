# Production Deployment of Multi-Tier Application on Kubernetes (VProfile Stack)

## 1. Project Overview

This project is all about taking our already packaged web application (the VProfile Java stack) and launching it safely onto a live production environment. We're using **Kubernetes** running on **Amazon Web Services (AWS)** for this.

Our main goal is simple: we need our application to be reliable, handle lots of traffic, and never go down. Kubernetes gives us the tools to ensure:

* **High Availability (HA):** Our app is always running.

* **Fault Tolerance:** The system fixes itself if something crashes.

* **Easy Scaling:** We can handle more users without breaking a sweat.

Since we define everything using Kubernetes files, our setup is **portable**. We can easily run this exact same application anywhere—from a local machine to any cloud—without complex changes.

---

## 2. Key Architectural Requirements

We designed this project to meet essential requirements that any solid production application needs.

| Requirement | What We Need | How Kubernetes Helps | 
| :----- | :----- | :----- | 
| **Always Running** | Containers must stay up; we need reliable servers (compute nodes). | If a server fails, Kubernetes automatically moves and restarts our containers. | 
| **Self-Healing** | If a container crashes or stops responding, it needs to fix itself. | Kubernetes constantly monitors the health of our app and replaces bad containers automatically. | 
| **Grow Easily** | We need to handle more users quickly. | We can adjust one setting (`replicas` count) in our setup files to instantly add more application copies. | 
| **Save Data** | The database must keep its data safe even if the container restarts. | We use secure AWS hard drives (**EBS Volumes**) connected through **Persistent Volume Claims (PVC)** to save our data. | 
| **Stay Secret** | Passwords (like the database password) must be hidden. | We use **Secret** objects to safely store sensitive information and inject it into the correct containers. | 

---

## 3. Technology Stack

Here are the main tools we are using to get this done:

| Category | Component | Purpose | 
| :----- | :----- | :----- | 
| **Orchestration** | Kubernetes | The brain that manages all our containers, scaling, and self-healing. | 
| **Infrastructure** | AWS (EKS/kops) | Our cloud provider, which hosts the Kubernetes cluster and provides storage (EBS). | 
| **Application** | VProfile Java Stack | Our multi-part application (Web, Database, Caching, and Messaging). | 
| **Backend Services** | MySQL, Memcached, RabbitMQ |
