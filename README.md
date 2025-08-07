# Cloud Computing


## What is Cloud Computing?

Cloud Computing means using the Internet to access services like Storage, Software, Server, Networking, etc., instead of keeping them on your personal computer or a company's physical data center. 

You can access these services from anywhere using an Internet connection. Cloud Computing is based on a **Pay-as-you-go** model.

## Example

Previously, you used to store your photos, videos, and other documents on a pen drive or HDD. With Cloud Computing, you store them on services like **Google Drive** or **iCloud** and can access them from anywhere on any device.

## Benefits

- **Faster Access**: Quick access to data and applications anytime.
- **Scalability**: Easily scale resources up or down based on demand.
- **Available Any Time**: Services are available 24/7.
- **Security**: Data is protected by robust cloud security measures.
- **Data Loss Prevention**: Cloud service providers back up your data to prevent loss.
- **Cost Saving**: No need to invest in expensive hardware; pay only for what you use.

## Before Cloud Computing vs After Cloud Computing

<img width="1536" height="1024" alt="ChatGPT Image Jun 21, 2025, 02_50_31 PM" src="https://github.com/user-attachments/assets/f66607ef-f923-4200-9e27-398fd06cf41e" />

---



| Aspect                | Before Cloud Computing                        | After Cloud Computing                          |
|-----------------------|----------------------------------------------|-----------------------------------------------|
| Storage               | Local devices like pen drives, HDDs          | Cloud storage accessible anywhere             |
| Accessibility         | Limited to the device where data is stored   | Accessible globally via Internet               |
| Cost                  | High upfront investment in hardware          | Pay-as-you-go model reduces upfront costs     |
| Scalability           | Difficult and expensive to scale              | Easy and flexible resource scaling             |
| Data Backup           | User responsibility with limited backup options | Automatic backup and data recovery            |
| Software Updates      | Manual updates needed on each device          | Automatic updates and maintenance by provider |
| Collaboration         | Limited or manual file sharing                 | Real-time collaboration and sharing            |
| Security              | Dependent on user’s own measures               | Advanced security provided by cloud vendors   |


## After Cloud Computing vs After Cloud Computing

| Feature                      | Description                                                                                              |
|------------------------------|----------------------------------------------------------------------------------------------------------|
| Global Accessibility         | Resources and applications hosted on global cloud data centers ensure access from virtually anywhere, anytime. |
| Elastic Scalability          | Infrastructure dynamically scales up or down based on demand, optimizing resource use and cost efficiency. |
| Managed Services             | Cloud providers offer fully managed databases, AI/ML platforms, container orchestration, serverless computing, and more, reducing operational overhead. |
| High Availability & Disaster Recovery | Built-in redundancy, geographic distribution, and failover mechanisms provide strong uptime guarantees and data protection. |
| Advanced Security            | Includes multi-layered security models, encryption at rest and in transit, compliance certifications, identity and access management, and continuous monitoring. |
| Automation & DevOps Enablement | Infrastructure-as-Code (IaC), CI/CD pipelines, and automated monitoring facilitate rapid development, deployment, and maintenance. |
| Pay-as-You-Go and Cost Optimization | Flexible pricing models let you pay based on actual usage with tools to monitor and optimize costs. |
| Collaboration & Integration | Cloud platforms integrate with a broad ecosystem of tools and services, enabling seamless collaboration and workflow automation. |

---

# Cloud Providers ☁️

## 1. AWS Cloud ☁️

Amazon Web Services (AWS) is a cloud platform providing services such as servers, storage, networking, machine learning, and more over the internet. 

- **Founded:** 2006  
- **First Services Launched:** S3 and EC2  
- **Current Market Value:** Around $1.8 - 2.0 trillion  
- **Q4 2024 Revenue:** $28.8 billion  
- **Customer Base:**  
  - Over 1 million active users worldwide  
  - Roughly 2.38 million companies  

## 2. Azure Cloud ☁️

Microsoft Azure is a cloud computing platform that enables building, running, and managing applications using Microsoft's global data centers without owning physical servers. 

- **Founded:** Announced October 2008; launched February 2010; renamed March 2014  
- **Current Market Value:** Microsoft’s overall valuation is around $3.1 trillion as of early 2025  
- **2023 Revenue (Intelligent Cloud segment including Azure):** $96.2 billion  
- **Customer Base:** Over 722 million Azure Active Directory users, approaching 1 billion users worldwide  

## 3. GCP (Google Cloud Platform) ☁️

Google Cloud Platform (GCP) provides tools and services to build, deploy, and manage applications and data on Google’s infrastructure—the same infrastructure behind Google Search, YouTube, and Gmail.

- **Founded:** First announced in 2008; publicly available around 2011  
- **2023 Revenue:** Approximately $43.2 billion  
- **2022 Revenue:** $26.3 billion (about 33% year-over-year growth)  
- **Customer Base:** Over 1.16 million companies  

## Cloud Computing Architecture 

<img width="1366" height="768" alt="Yellow Colorful UI Health and Wellness Bio-Link Website (1)" src="https://github.com/user-attachments/assets/6ab9e863-6e8f-4db7-98f9-8ca2b21e4d44" />

Cloud Computing Architecture is the structural blueprint that defines how cloud computing components and services work together. It combines front-end (client-side) and back-end (server-side) systems to deliver cloud services such as IaaS, PaaS, and SaaS.

### 1. **Front-End (Client Side)**

This is the part users interact with the help of component’s 

Components 

- **Web Browser or Client Application**: Used to access cloud services.
- **User Interface (UI)**: Web-based dashboard, mobile app, or custom tool.

### 2.Network

The **network** acts as the *bridge* between the **client (user)** and the **cloud service provider**. It plays a critical role in ensuring data is transmitted reliably, securely, and quickly between both ends.

### 3. **Back-End (Cloud Provider Side)**

The **Back-End** is the heart of cloud computing. It consists of all the systems and services that store, process, and manage your data behind the scenes. These are run and maintained by **cloud service providers** like AWS, Azure, and GCP.

### Components 

### 1. **Application**

- **What it means:** Cloud-hosted applications that users access through the internet.
- **Examples:**
    - Gmail, Google Docs, Salesforce, Zoom, Netflix
- **In the cloud:** These applications run on remote servers and are accessed through web browsers or APIs, eliminating the need for local installation.

### 2. **Service**

- **What it means:** Core cloud functions that perform specific tasks.
- **Types of Services:**
    - **Compute Services:** Virtual Machines (e.g., AWS EC2, Azure VM, GCP Compute Engine)
    - **Database Services:** Manage structured/unstructured data (e.g., AWS RDS, Firebase, Azure Cosmos DB)
    - **AI/ML Services:** Pre-built machine learning tools (e.g., AWS SageMaker, Azure ML, GCP Vertex AI)
    - **DevOps Tools:** CI/CD pipelines, monitoring, IaC

### 3. **Storage**

- **What it means:** Where your data is saved in the cloud.
- **Types of Cloud Storage:**
    - **Object Storage:** AWS S3, Azure Blob, GCP Cloud Storage
    - **Block Storage:** For databases and OS disks (e.g., AWS EBS)
    - **File Storage:** Shared file systems (e.g., Amazon EFS, Azure Files)

### 4. **Security**

- **What it means:** All layers of protection to keep data and applications safe.
- **Includes:**
    - **IAM (Identity & Access Management):** Controls who can access what
    - **Encryption:** Protects data at rest and in transit
    - **Firewalls & DDoS Protection:** Defend against attacks
    - **Compliance Controls:** Meet global standards like ISO, HIPAA, GDPR

### 5. **Runtime**

- **What it means:** The environment in which code/applications run.
- **Includes:**
    - **Operating Systems (Linux, Windows)**
    - **Containers & Kubernetes**
    - **Function-as-a-Service (FaaS):** Like AWS Lambda or Azure Functions


### Cloud Service Models (IaaS, PaaS, SaaS)

 <img width="1280" height="1016" alt="diagram-export-15-06-2025-19_34_37" src="https://github.com/user-attachments/assets/6b257ca7-3f0f-4301-8f0b-2487be253c95" />

 # 1. 🧱 What is **IaaS ?**

IaaS Stands for **Infrastructure as a Service it. is a Cloud service Model that Delivers Computing Resources over the Internet** 

It allows user or organisations to rent the IT infrastructure such as Server , Storage , Network etc 

## IaaS Components

| Resource | Purpose |
| --- | --- |
| Compute (VMs) | Run applications or software |
| Storage | Store files, databases, backups |
| Networking | Connect services and users |
| Load Balancers | Distribute traffic |
| Firewalls/Security | Secure your cloud resources |

### 🔍 **Examples of IaaS Providers:**

- **Amazon EC2** (AWS)
- **Google Compute Engine** (GCP)
- **Microsoft Azure Virtual Machines**
- **DigitalOcean Droplets**
- **IBM Cloud Infrastructure**

### **Best Use Cases:**

- Hosting websites and applications
- Testing and development environments
- Backup and disaster recovery
- Running custom enterprise software
- Big data analysis & batch processing

## 🧱 **IaaS Advantages**

| No | Advantage | Explanation |
| --- | --- | --- |
| 1️⃣ | Cost-Effective | No need to buy physical servers — pay only for what you use (pay-as-you-go). |
| 2️⃣ | Scalable | Easily scale up/down based on your traffic or demand — perfect for startups and enterprises. |
| 3️⃣ | On-Demand Resources | Get virtual machines, storage, and networking instantly when needed. |
| 4️⃣ | Disaster Recovery & Backup | Built-in support for secure backups and quick recovery. |
| 5️⃣ | Flexible & Customizable | You choose OS, runtime, and app stack — complete control over your setup. |
| 6️⃣ | Global Access | Deploy applications and data across multiple regions for better performance. |
| 7️⃣ | Quick Deployment | Launch infrastructure in minutes instead of waiting for hardware setup. |
| 8️⃣ | Enhanced Security | Leading cloud providers offer enterprise-level firewalls, encryption, IAM. |
| 9️⃣ | Supports Innovation | Developers and DevOps teams can test and experiment quickly. |
| 🔟 | No Maintenance Burden | Providers handle hardware updates, power, cooling, and physical security. |

## ❌ **Limitations of IaaS (Infrastructure as a Service)**

| No | **Limitation** | **Explanation** |
| --- | --- | --- |
| 1️⃣ | **High Responsibility** | Users are responsible for managing OS, middleware, runtime, and apps. |
| 2️⃣ | **Complexity for Beginners** | Requires technical knowledge to configure, secure, and maintain infrastructure. |
| 3️⃣ | **Security Risks** | Misconfigured servers or networks can lead to vulnerabilities or attacks. |
| 4️⃣ | **Scaling Requires Expertise** | Although scalable, auto-scaling and load balancing need to be set up manually. |
| 5️⃣ | **Downtime Risk** | Poor infrastructure design or unmonitored resources can cause app failures. |
| 6️⃣ | **Cost Management Challenges** | Without monitoring and optimization, costs can spiral due to pay-as-you-go billing. |
| 7️⃣ | **Vendor Lock-In Possibility** | Moving workloads between IaaS providers can be difficult due to platform-specific configurations. |
| 8️⃣ | **Limited Automation by Default** | Users must implement their own automation (e.g., for backups, deployments). |

# 🌐 What is **PaaS (Platform as a Service)?**

**PaaS** provides a **ready-to-use platform** where developers can **build, test, deploy, and manage applications** — without worrying about managing the underlying infrastructure like servers, storage, or operating systems.

It Provides complete Development and Deployment. Environment in the Cloud. It is a ready made platform where you can focus on building your application without worrying about the infrastructure 

PaaS Components 

| Component | Purpose |
| --- | --- |
| Runtime Environment | Execute your app (e.g., Java, Python) |
| Development Tools | IDEs, Git, CI/CD pipelines |
| Database Services | SQL/NoSQL databases |
| App Hosting | Easily deploy and host web/mobile apps |
| Middleware | Connects app logic to infrastructure |

### 🔍 **Examples of PaaS Providers:**

- **Google App Engine**
- **AWS Elastic Beanstalk**
- **Microsoft Azure App Service**
- **Heroku**
- **Red Hat OpenShift**

### **Best Use Cases:**

- Rapid app development
- Microservices and container-based apps
- Mobile and web applications
- Continuous Integration/Delivery (CI/CD)

## ❌ **Limitations of PaaS**

| No | **Limitation** | **Explanation** |
| --- | --- | --- |
| 1️⃣ | **Less Control Over Infrastructure** | You don’t manage the OS or hardware — limits deep customization. |
| 2️⃣ | **Vendor Lock-In** | Hard to migrate your app to another PaaS provider due to proprietary tools/APIs. |
| 3️⃣ | **Limited Runtime Support** | Only specific languages or frameworks may be supported (e.g., Node.js, Java). |
| 4️⃣ | **Scalability May Be Limited** | Automatic scaling is helpful, but not always flexible for complex enterprise needs. |
| 5️⃣ | **Security Concerns** | Shared environments can raise data privacy or compliance issues. |
| 6️⃣ | **Integration Challenges** | Integrating with on-premise systems or external services might be complex. |
| 7️⃣ | **Performance Issues** | Since resources are shared, spikes in usage from other tenants may impact performance. |
| 8️⃣ | **Hidden Costs** | Additional charges may apply for premium features, bandwidth, or scaling. |

## 🛠️ **Advantages of PaaS (Platform as a Service)**

| No | Advantage | Explanation |
| --- | --- | --- |
| 1️⃣ | Faster Development | Developers can focus on writing code, not managing infrastructure — speeds up app development. |
| 2️⃣ | Reduced Complexity | No need to worry about OS updates, servers, or networking — the provider handles it. |
| 3️⃣ | Cost-Efficient | No need to invest in physical hardware or manage large DevOps teams — saves cost. |
| 4️⃣ | Built-in Tools | Comes with pre-integrated tools like CI/CD pipelines, databases, monitoring, and version control. |
| 5️⃣ | Automatic Scaling | Apps automatically scale up/down based on user traffic or resource demands. |
| 6️⃣ | Easy Deployment | Deploy apps with just a few clicks or commands — no manual setup required. |
| 7️⃣ | Cross-Platform Support | Easily develop apps for web, mobile, or desktop — all in one place. |
| 8️⃣ | Collaboration-Friendly | Multiple developers can work on the same app with shared environments. |
| 9️⃣ | Security & Compliance | Providers manage and regularly update security, patching, and compliance standards. |
| 🔟 | Focus on Innovation | Developers spend more time creating features rather than handling infrastructure. |

# 📦 What is **SaaS (Software as a Service)?**

**SaaS** is a cloud service model where **software applications are delivered over the internet**. You don’t need to install, maintain, or update anything — you just log in and use the app via your browser or mobile device.

**SaaS** is a cloud computing model where **software is hosted centrally by a cloud provider** and accessed by users via a web browser or app. Everything — including the infrastructure, application, runtime, security, and maintenance — is handled by the provider.

## 📦 **SaaS Components**

| # | Component | Description |
| --- | --- | --- |
| 1️⃣ | Application Software | The user-facing app (frontend & backend) that delivers core functionality. |
| 2️⃣ | Multitenant Architecture | A single app instance serves multiple users with isolated, secure data. |
| 3️⃣ | Data Management Layer | Manages storage, access, backup, and integrity of user data. |
| 4️⃣ | APIs | Allow integration with third-party services and internal tools. |
| 5️⃣ | User Management | Handles user registration, login, roles, and permissions. |
| 6️⃣ | Billing & Subscription | Manages pricing plans, payments, invoices, and trials. |
| 7️⃣ | Monitoring & Analytics | Tracks performance, errors, usage, and user behavior. |
| 8️⃣ | Security Layer | Implements encryption, authentication, compliance (e.g., GDPR, SOC2). |
| 9️⃣ | DevOps & CI/CD Pipeline | Automates deployment, updates, and scalability of the SaaS platform. |
| 🔟 | Customer Support Tools | Provides in-app help, ticketing, chat support for user assistance. |

## 📋 **Examples of SaaS:**

| Category | Examples |
| --- | --- |
| **Email** | Gmail, Outlook 365 |
| **Storage** | Dropbox, Google Drive |
| **CRM** | Salesforce, Zoho CRM |
| **Collaboration** | Slack, Notion, Microsoft Teams |
| **Video Conferencing** | Zoom, Google Meet |
| **Design** | Canva, Figma |
| **Finance/HR** | FreshBooks, QuickBooks, Workday |

## Advantages of SaaS:

| No | Advantage | Explanation |
| --- | --- | --- |
| 1️⃣ | No Setup Hassle | Just sign up and start using — no hardware needed |
| 2️⃣ | Low Cost | Pay for what you use — great for startups and small businesses |
| 3️⃣ | Always Updated | Get the latest features automatically |
| 4️⃣ | Accessibility | Use apps on web or mobile from anywhere |
| 5️⃣ | Easy Collaboration | Multiple users can work together in real-time |
| 6️⃣ | High Availability | Cloud providers offer 99.9% uptime or more |
| 7️⃣ | Security & Backups | SaaS providers manage encryption, backups, and disaster recovery |
| 8️⃣ | Fast Deployment | No need to wait for long installations or configuration |

### ❌ **Limitations of SaaS (to be aware of):**

- Less control over features/customization
- Dependency on internet connection
- Data privacy concerns (if sensitive data is stored on third-party servers)

### 🧪 **Best Use Cases for SaaS:**

- Startups and small businesses avoiding IT complexity
- Remote teams and distributed workforces
- Enterprises using CRM, HR, and collaboration tools
- E-learning platforms and customer support portals

# Cloud Deployment Models

<img width="562" height="782" alt="diagram-export-15-06-2025-21_56_28" src="https://github.com/user-attachments/assets/40509d56-1cf0-4343-9a7f-f2975aa7c887" />


## ☁️ **What are Cloud Deployment Models?**

Cloud deployment models define **how cloud services are made available to users** — based on **ownership, size, access, and purpose**.

They describe **where the infrastructure is hosted** and **who controls it**.

📊 **Types of Cloud Deployment Models**

| **Model** | **Description** |
| --- | --- |
| 🌐 **Public Cloud** | Services are offered over the internet to anyone. Owned and operated by third-party providers like AWS, Azure, GCP. |
| 🏢 **Private Cloud** | Used exclusively by a single organization. Hosted either on-premise or by a third-party. |
| 🌤️ **Hybrid Cloud** | A combination of public and private cloud. Allows data/apps to move between the two. |
| 🧠 **Community Cloud** | Shared by organizations with common goals (e.g., government, healthcare consortiums). |

## ☁️ **1. Public Cloud**

A **Public Cloud** is a cloud computing model where services (like storage, compute, networking) are **offered over the internet** by **third-party providers** (like AWS, Microsoft Azure, or Google Cloud).

The public deployment model refers to **a cloud computing model where IT services are hosted and delivered by a third party and accessed over the Internet**. In this model, services are available to the "public," meaning any organization or end user can create an account and use the services.

This model **operates on a pay-as-you-go basis, where users only pay for the resources they use**

### 🏢 **Used By:**

- Startups
- Developers and testers
- Businesses looking to scale quickly

### 📦 **Example Services:**

- AWS EC2, S3
- Google Cloud Storage
- Microsoft Azure App Services

Advantage 

✅ Cost-effective

✅ Scalable

✅ Easy setup

✅ No maintenance required

Disadvantage 

❌ Less control over data

❌ Potential security/compliance concerns

## 🏢 2. **Private Cloud**

The private cloud deployment model refers to **a dedicated environment for a single organization, where resources are not shared with other organizations**

This model offers greater control and flexibility over cloud resources, as the organization is the sole owner and has complete command over service integration, IT operations, policies, and user behavior.

Additionally, the private cloud provides better data security and privacy, making it suitable for organizations with strict compliance requirements.

However, it can be more expensive to set up and maintain compared to public cloud models.

### 🏢 **Used By:**

- Banks and financial institutions
- Healthcare organizations
- Government agencies

### 📦 **Example Technologies:**

- VMware vSphere
- OpenStack Private Cloud
- Microsoft Azure Stack

### Advantages

✅ High security

✅ Full control

✅ Customizable

✅ Better compliance

### Disadvantages

❌ Expensive to build & maintain

❌ Requires skilled staff

## 🌤️3  **Hybrid Cloud**

A hybrid cloud deployment model **combines the use of both public and private cloud environments**, allowing organizations to leverage the benefits of each while maintaining flexibility and control. This model enables workloads to be deployed across different environments, such as on-premises data centers, private clouds, and public clouds, and can also include edge locations.

The hybrid cloud approach allows organizations to run applications and store data in the most suitable environment based on factors like security, compliance, performance, and cost.

One of the key features of a hybrid cloud is its ability to integrate and orchestrate resources across different environments, ensuring seamless data and process sharing between public and private clouds.

### 🏢 **Used By:**

- Enterprises needing flexibility
- Organizations with regulatory needs but also public traffic (e.g., e-commerce)

### 📦 **Example Setup:**

- Use a private cloud for storing customer data.
- Use AWS or Azure public cloud for hosting websites or mobile apps.

## Advantages

✅ Flexibility

✅ Cost-effective scaling

✅ Better disaster recovery

✅ Sensitive data stays private

## Disadvantage

❌ Complex integration

❌ Inconsistent security policies

❌ Requires expert management

## 🧠 **4. Community Cloud**

A **Community Cloud** is a cloud infrastructure **shared by several organizations** with **common goals, compliance needs, or industries**.

The cloud environment is generally built to help them achieve that purpose or goal.

This model combines the privacy and control of a private cloud with the collaborative environment of a public cloud.

The infrastructure, and hence the cost, is shared, allowing for the purchase of infrastructure that any single member organization may not have been able to afford.

a community cloud offers better security and privacy than a public cloud, sharing between several organizations. This means that it could be hard to handle security issues and settle on service-level agreements (SLAs) when sharing data and resources with other participants.

### 🏢 **Used By:**

- Government departments
- Universities or research institutions
- Healthcare consortiums

### 📦 **Example Use Case:**

A group of hospitals share a community cloud to store and manage electronic health records (EHRs) with common security requirements.

### Advantages

✅ Cost-sharing

✅ Collaboration-friendly

✅ Aligned compliance & policy needs

### Disadvantage

❌ Shared control

❌ Limited scalability

❌ Complex to manage governance
