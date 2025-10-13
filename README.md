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
| **Backend Services** | MySQL, Memcached, RabbitMQ | The essential components the web app needs to run. | 
| **Networking** | ClusterIP Service, Nginx Ingress Controller, AWS ALB | Tools for internal communication and getting outside traffic to our app. | 

---

## 4. Kubernetes Architecture Breakdown

We break the application into small, manageable pieces using standard Kubernetes objects.

### A. Core Services (Deployments & Services)

Every part of our application (Tomcat, MySQL, Memcache, RabbitMQ) is managed by a **Deployment**. Deployments ensure the right number of copies are running at all times.

* **Tomcat Deployment:** Runs our main VProfile web application.

* **Backend Deployments:** Run the MySQL, Memcache, and RabbitMQ containers.

  * **Service (`ClusterIP`):** Since containers (Pods) in Kubernetes change IP addresses often, we use a stable internal address (like a load balancer) called a `ClusterIP` Service. For example, the Tomcat app connects to a stable name like `vprodb:3306`, and the Service handles routing the request to the correct database container.

### B. Stateful Management (MySQL Persistence)

The MySQL database needs to store data (it's a **Stateful** app). We handle this by separating the storage from the container:

* **PersistentVolumeClaim (PVC):** This is essentially a request we write in a file that says, "I need 10 GB of storage."

* **StorageClass (AWS EBS):** This acts like a driver. It takes the PVC request, goes to AWS, automatically creates a new hard drive (**EBS volume**), and attaches it to the cluster.

* **Volume Mount:** We configure the MySQL container to save all its data directly onto that external EBS volume. If the container is deleted, the data is safe.

### C. Security and Configuration

* **Secrets:** We use a `Secret` object to securely store passwords for MySQL and RabbitMQ. This prevents us from leaving passwords in plain text in our definition files. The passwords are then safely given to the correct containers when they start.

### D. External Access and Routing

To let users access our web application from the internet:

1. **Ingress Controller (Nginx):** This manages the outside traffic coming into our cluster.

2. **Ingress Resource:** We write a rule here that tells the system what to do with incoming requests.

   * The Ingress Controller automatically creates an **AWS Application Load Balancer (ALB)** for us.

   * The **Ingress Rule** says: "If a user visits `vprofile.mywebsite.com`, send that traffic to the internal **Tomcat Service**."

---

## 5. Critical Implementation Details

This section covers special configurations we used to solve production challenges:

### Node Affinity for EBS Volumes

AWS hard drives (**EBS volumes**) are tied to specific geographical zones (e.g., `us-east-1a`). The database container **must** run on a server (worker node) in the *exact same zone* as its hard drive.

* **Solution:** We put special labels (like sticky notes) on our worker nodes with their zone names. Then, the database deployment file uses a **Node Selector** to ensure the database pod is only ever scheduled on a server that has the matching zone label.

### EBS Volume Cleanup (`lost+found`)

When a new hard drive is attached to a Linux system, it often has a folder called `lost+found`. If the MySQL service sees this folder when it first starts up, it might fail.

* **Solution:** We use an **`initContainer`** inside the database pod. This tiny container runs *first*, mounts the volume, and deletes the `lost+found` folder, ensuring the volume is perfectly empty before the main MySQL container starts.

### Application Initialization Delay

The frontend application (Tomcat) cannot start successfully until the backend services (like the database and RabbitMQ) are fully booted and ready.

* **Solution:** We add an **`initContainer`** to the Tomcat deployment. This container is configured to check the network status of the backend services repeatedly. It acts as a gate—the main Tomcat application container only launches once all backend checks pass.

---

## 6. Organizational and Business Benefits

Adopting this Kubernetes-based architecture provides tangible benefits that translate into significant organizational savings and risk mitigation:

### A. Cost Savings Through Efficiency (FinOps)

| Benefit | Impact on Budget | 
| :----- | :----- | 
| **Eliminate Over-Provisioning** | Kubernetes allows you to specify the exact CPU and memory **requests** for each container. This precision minimizes idle resources, ensuring you only pay AWS for the compute capacity the application actively needs, leading to direct savings on cloud computing costs. | 
| **Right-Sizing Infrastructure** | The cluster can be configured with auto-scaling to add or remove worker nodes based on aggregate demand. This dynamic scaling avoids the cost of maintaining expensive, high-capacity servers 24/7/365 when demand is low. | 
| **Operational Automation** | Kubernetes handles automated deployment, rollback, and self-healing (restarting failed containers). This frees up high-cost Senior DevOps/SRE personnel from routine monitoring and manual intervention, allowing them to focus on innovation and higher-value tasks. | 

### B. Revenue Protection and Risk Mitigation

| Benefit | Impact on Business Continuity | 
| :----- | :----- | 
| **Zero Downtime (High Availability)** | Built-in replication and self-healing ensure that if any component (container, server, or even an AWS Availability Zone) fails, traffic is instantly rerouted to a healthy replica. This virtually eliminates catastrophic downtime, protecting customer access and revenue streams. | 
| **Improved Customer Trust** | Consistent, 24/7 availability meets stringent Service Level Agreements (SLAs) and maintains a positive brand reputation. Uninterrupted service directly leads to higher customer satisfaction and loyalty. | 
| **Data Integrity and Faster Recovery** | By using Persistent Volume Claims (PVC) tied to AWS EBS, the database data is decoupled from the compute instance. In case of a database crash, the recovery is near-instantaneous, requiring only a container restart, without the delay and risk associated with restoring from backup. | 

### C. Strategic Flexibility

| Benefit | Impact on Long-Term Strategy | 
| :----- | :----- | 
| **Mitigating Vendor Lock-in** | Since the application is packaged using open-source containers and orchestrated with Kubernetes, the deployment artifact (`.yaml` files) is cloud-agnostic. Your organization retains the flexibility to shift workloads to another cloud provider (GCP, Azure) or an on-premise private cloud if AWS raises prices, providing crucial negotiation leverage. | 
| **Rapid Feature Deployment** | The standardized nature of Kubernetes pipelines (CI/CD) allows developers to deploy new application versions rapidly and reliably, reducing the time-to-market for new features and bug fixes. |

![alt text](https://github.com/Tejesh1105/Kubernetes-Application-Deployment-Project-/blob/main/Kubernetes%20app%202.PNG)


