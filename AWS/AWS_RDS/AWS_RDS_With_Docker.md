# AWS RDS With Docker 

---

### Pre-requisites
- RDS
- EC2
- Docker


---

Step 1. Go to AWS RDS Dashboard and click on Create Database
<img width="3264" height="1901" alt="image" src="https://github.com/user-attachments/assets/7c17d294-583a-41d0-a1f4-919ef885bee8" />


---

Step 2. Choose a database creation method and Choose Engine options In case we use MySQL 
<img width="3326" height="1952" alt="image" src="https://github.com/user-attachments/assets/a5448a74-94b5-47e1-a831-e6f686c7a5c5" />

---


Step 3. Choose Templates Free tier for Practice Purpose 
<img width="1" height="1" alt="image" src="https://github.com/user-attachments/assets/2e6a3565-a42b-49f8-8a93-626e3a050f27" />


---

Step 4. Fill the all information 
<img width="3340" height="1886" alt="image" src="https://github.com/user-attachments/assets/75a13781-daa4-414d-af5f-8e81643ef7de" />
<img width="3267" height="1974" alt="image" src="https://github.com/user-attachments/assets/f70377ab-e5bd-4898-bedf-d22d9a2aee4d" />


---

Step 5. Connectivity and Scroll down and click on Create Database and wait 9-10 minutes 
<img width="3327" height="1931" alt="image" src="https://github.com/user-attachments/assets/ab703e25-e802-4d90-a819-5e292763611d" />
<img width="2684" height="931" alt="image" src="https://github.com/user-attachments/assets/75e64985-aaba-4226-8735-b8941012e4ba" />


---

Step 6. After Successfully creating RDS Database first change the Security Groups inbound rules Anywhere IP for access database using the ec2 
<img width="3219" height="1212" alt="image" src="https://github.com/user-attachments/assets/edc44e95-6f57-49a9-8fc9-8b90cf8ac914" />


---

Step 7. Create a EC2 and Connect with your RDS Database Using This Commands

```bash
mysql -h <Add RDS Endpoint > \
  -P 3306 \
  -u <your Database user name> \
  -p
```
**after -p it will ask you for database password**

---

Step 8. Now Install Docker 
```bash
sudo apt-get install docker.io -y
sudo usermode -aG docker $USER
```
---

Step 9. Now Pull the image form docker hub

```docker pull philippaul/node-mysql-app:02```

---

Step 10. Attach your RDS Database with this docker image 

```bash
docker run --rm -d -p 80:3000 \
  -e DB_HOST="Your RDS Endpoint " \
  -e DB_USER="Database Username " \
  -e DB_PASSWORD="Database password" \
  -e DB_PORT="3306" \
  philippaul/node-mysql-app:02
```


---

Step 11. Copy your EC2 Public IP and Paste on new tab and access the application add your data 
<img width="3209" height="1392" alt="image" src="https://github.com/user-attachments/assets/0430dc46-47eb-40e8-8a16-06e573c4233f" /> 

---

Step 12. Access the data on ec2 using Step 7
<img width="1931" height="1170" alt="image" src="https://github.com/user-attachments/assets/7d06f239-440a-43d6-a1f5-2e10823dcb73" />
<img width="1124" height="993" alt="image" src="https://github.com/user-attachments/assets/73df79b8-5f96-45a1-ae04-d055724747cd" />

---
> ⚠️ Please delete all resource after practice 
