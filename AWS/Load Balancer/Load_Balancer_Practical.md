<img width="1190" height="685" alt="Screenshot 2025-08-13 at 3 03 22 PM" src="https://github.com/user-attachments/assets/4e01a2df-731f-453f-a2c3-f9b5d0d659d9" />


---

# AWS Load Balancer Practical — Setup Guide

## 🏗 Architecture Overview
- **2 EC2 instances** (Amazon Linux 2 recommended)
- **Security Group** allowing inbound: 22, 80, 81, 443
- **Web servers**:
  - Server A: Nginx (listens on 80)
  - Server B: Apache httpd (listens on 81 or 80)
- **Application Load Balancer (ALB)**
- **Target Groups**:
  - TG-Nginx → port 80
  - TG-Httpd → port 81
- **ALB Listeners and Rules**:
  - Listener :80 → forward to target groups
  - Optional :443 → HTTPS with ACM Certificate

---

## ✅ Prerequisites
- AWS account with IAM permissions for EC2, ELB, ACM
- A VPC with at least **2 public subnets across different AZs or you can use Default**
- A key pair for SSH access
## 1️⃣ Create Security Group

**Name:** `lb-practical-sg`

**Inbound Rules:**
| Port | Protocol | Source |
|------|----------|--------|
| 22   | TCP      | Your IP |
| 80   | TCP      | 0.0.0.0/0 |
| 81   | TCP      | 0.0.0.0/0 |
| 443  | TCP      | 0.0.0.0/0 |


<img width="3197" height="963" alt="image" src="https://github.com/user-attachments/assets/07b13be5-a047-4a36-929d-80462fc14eb2" />

---

**Outbound Rules:**
- Allow All Traffic: `0.0.0.0/0`

Attach this Security Group to **both EC2 instances** and the **ALB**.

## 2️⃣ Launch 2 EC2 Instances with Security groups which you created 

**Example names:**
- `ec2-nginx-a`
- `ec2-httpd-b`


## 3️⃣ Install and Configure Web Servers in both EC2
```bash
sudo yum update -y
sudo install nginx httpd -y
```
## 4️⃣ Now we need to change port number of one severe (eg nginx)

```bash
sudo vim /etc/nginx/nginx.conf
```

<img width="2410" height="1541" alt="image" src="https://github.com/user-attachments/assets/70ae49bd-d29c-42e3-8d5d-7848993ea4aa" />

---

```bash
systemctl restart nginx
systemctl enable nginx
```

## 5️⃣ Add index.html file in httpd html location use my servre html files for demo.

```bash
cd /var/www/html
sudo vim index.html
```
## 6️⃣ Start the Server 

```bash
systemctl start httpd
systemctl enable httpd
```

# Now Time to create Load Balancer 
- Step 1. Go to Load Balancer 

- Step 2. Create Target Groups (One for Httpd and One for nginx)
<img width="3325" height="1927" alt="image" src="https://github.com/user-attachments/assets/0fbcfe15-9ab2-4684-a83a-1febfda6ed1b" />

Step 3. 
<img width="3329" height="1489" alt="image" src="https://github.com/user-attachments/assets/c3598b07-0c7d-4a56-b913-92a274d86d39" />

<img width="2706" height="889" alt="image" src="https://github.com/user-attachments/assets/ca26866b-e93f-4ef3-8815-10d52b0d1fce" />

---

## Create another  for nginx with port number 81
<img width="2409" height="894" alt="image" src="https://github.com/user-attachments/assets/06bd632e-ceab-4c4d-9208-7e95638e0700" />

- Select and Click add Include as pending below

<img width="2939" height="1070" alt="image" src="https://github.com/user-attachments/assets/cdc384b7-632c-4463-b885-667d87551a28" />

---

## target
<img width="3330" height="1010" alt="image" src="https://github.com/user-attachments/assets/e6f256d7-0194-4618-8c6e-56a25b2cb06d" />

---

## Create Load Balancer 
<img width="3316" height="1574" alt="image" src="https://github.com/user-attachments/assets/efede10c-9d5d-441e-9dfa-8034fbabbc8c" />

- Select

<img width="3271" height="1336" alt="image" src="https://github.com/user-attachments/assets/33cd0e46-8f50-426b-bc52-27799d712269" />

- Add Security Group which you created
  
<img width="2992" height="524" alt="image" src="https://github.com/user-attachments/assets/ff6e3a09-a293-4f95-889a-6a1c494d7856" />

---

## Add targets group in Load Balancer 

<img width="3211" height="1384" alt="image" src="https://github.com/user-attachments/assets/daed276c-64a5-4866-b381-af814d05c872" />

<img width="3336" height="948" alt="image" src="https://github.com/user-attachments/assets/ce4e5017-0152-4e8b-9169-d516910985e0" />


---

# Output 
- Server 1
<img width="3338" height="1904" alt="image" src="https://github.com/user-attachments/assets/cde62844-98a5-45b7-9bc8-3cb7fad4e3bb" />

- After Refresh
- server 2
<img width="3318" height="1986" alt="image" src="https://github.com/user-attachments/assets/e8d4c2bf-607d-4e74-95f3-c071a28e902b" />

---

# Nginx on is running on  :81

<img width="3285" height="797" alt="image" src="https://github.com/user-attachments/assets/839ed5fb-de2e-4700-a0d7-9dc5f262ef41" />

---

## 🧹 Clean Up
- Delete ALB
- Delete Target Groups
- Terminate EC2 instances
- Delete Security Group

  # Do like if its help you 😊
