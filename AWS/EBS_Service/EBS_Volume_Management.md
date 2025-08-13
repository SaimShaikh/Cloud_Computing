# AWS EBS Volume Management on Linux

<img width="1218" height="626" alt="Screenshot 2025-08-09 at 4 58 31 PM" src="https://github.com/user-attachments/assets/6d7793c3-e50a-48c6-b5cc-f1425283a3f7" />

---

This guide explains how to **create**, **attach**, **format**, **mount**, and **move** an Amazon **EBS Volume** between EC2 instances running Amazon Linux.

---

## 1. Prerequisites
- AWS account
- At least **two EC2 instances** running Amazon Linux (in the **same Availability Zone** as your EBS volume)
- IAM permissions for EC2 & EBS operations
- SSH access to both EC2 instances

---

## Step-by-Step Procedure

### **Step 1 — Create EBS Volume**
1. Open **AWS Management Console → EC2 → Volumes → Create Volume**.

2. Choose:
   - **Size**: 10 GB  
   - **Availability Zone**: Same as your EC2 instance  
   - **Volume Type**: gp3 (or gp2)
     
3. Click **Step 1 Create Volume**.

---

### **Step 2 — Attach Volume to EC2**
1. Select your newly created volume.
2. Click **Actions → Attach Volume**.
3. Choose the target EC2 instance.
4. Note the **device name (EC2)** AWS assigns (e.g., `/dev/xvdbb`).

---

### **Step 3 — Identify Volume in EC2**
Connect with your EC2 instance:


```bash
ssh -i mykey.pem ec2-user@<EC2_PUBLIC_IP>
```
```bash
lsblk
```


<img width="456" height="103" alt="Screenshot 2025-08-09 at 5 22 16 PM" src="https://github.com/user-attachments/assets/be257c0a-1a2b-41d2-82cb-bdf49032cd44" />




```bash
sudo mkfs.ext4 /dev/nvme1n1 (⚠ Warning: Formatting erases all existing data.
Do not run this if the volume already has files you need.)
sudo mkdir /<any directory>
cd //<your directory created>
sudo touch file1.txt file2.txt
ls
```


## Step 4 — Unmount the Volume


```bash
cd ~
sudo umount /<your directory created>
```


## Step 5 — Attach to Another EC2
In AWS Console → Detach Volume from the first EC2.

Attach it to the second EC2 (same AZ).

## Step 6 — Mount on Second EC2
On the second EC2:

```bash
ssh -i mykey.pem ec2-user@<SECOND_EC2_IP>
lsblk
sudo mkdir /<same directory which you created in old ec2>
sudo mount /dev/nvme1n1 /<your directory created>
cd /<your directory created>
```

