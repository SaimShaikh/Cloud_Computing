# RDS MySQL + Dockerized App — “Student Records” Practical

- A step-by-step guide to:

- connect from EC2 to an AWS RDS MySQL 8.0 instance

- create a student_records database and a students table

- run a Dockerized app that connects via environment variables and verify data end-to-end


---

1) Prerequisites

- EC2 Linux host with Docker & MySQL client:

```
sudo apt-get update -y
sudo apt-get install -y mysql-client
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
sudo newgrp docker

```

2) Creates a RDS MySQL instance reachable from your EC2 (Security Group inbound on TCP 3306 from EC2’s SG or IP).
   - RDS details:
   - Host: your rds endpoint 
   - Port: 3306
   - User: admin
   - Password: <YOUR_DB_PASSWORD> ← store safely

3) 
- Pull the Docker image for app
 ``docker pull saime022/mystd2:latest``

4) Verify connectivity to RDS
   ```
     mysql -h YOUR RDS endpoint  \ -u admin -p'<YOUR_DB_PASSWORD>' -P 3306
   ```
   <img width="3360" height="2100" alt="image" src="https://github.com/user-attachments/assets/8f5487d6-4e40-41c7-a52f-5eb488ecd4d0" />

5) Inside MySQL:

``SHOW DATABASES;``
<img width="2047" height="1079" alt="image" src="https://github.com/user-attachments/assets/4e6f5321-45f3-42bd-bc51-47cace27e4b0" />

6) Create the students table (interactive)
```bash
CREATE TABLE IF NOT EXISTS students (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(100) NOT NULL,
  middle_name VARCHAR(100) NULL,
  last_name VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  date_of_birth DATE NOT NULL,
  current_location VARCHAR(255) NULL,
  phone VARCHAR(50) NULL,
  email VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_email (email),
  INDEX idx_name (last_name, first_name),
  INDEX idx_location (current_location)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```
<img width="3260" height="1279" alt="image" src="https://github.com/user-attachments/assets/a26cfc39-ed43-48e7-b970-5b761b27d5e5" />

**or If you prefer a single command from the shell**
```bash
mysql -h YOUR RDS ENDPOINT \
  -u admin -p'<YOUR_DB_PASSWORD>' -P 3306 <<'EOF'
CREATE DATABASE IF NOT EXISTS student_records;
USE student_records;
CREATE TABLE IF NOT EXISTS students (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(100) NOT NULL,
  middle_name VARCHAR(100) NULL,
  last_name VARCHAR(100) NOT NULL,
  age INT NOT NULL,
  date_of_birth DATE NOT NULL,
  current_location VARCHAR(255) NULL,
  phone VARCHAR(50) NULL,
  email VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_email (email),
  INDEX idx_name (last_name, first_name),
  INDEX idx_location (current_location)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
EOF

```

``DESCRIBE students;``
<img width="3360" height="2100" alt="image" src="https://github.com/user-attachments/assets/757b62c6-e0c4-4291-87d8-9cf9884cca7d" />

7) Run the Dockerized app
```
docker run --rm -d -p 3000:3000 \
  -e DB_HOST="YOUR RDS ENDPOINT" \
  -e DB_USER="admin" \
  -e DB_PASSWORD='<YOUR_DB_PASSWORD>' \
  -e DB_PORT="3306" \
  -e DB_NAME="student_records" \
  --name student-app \
  saime022/mystd2:latest
```
8) ``docker ps # PORTS should show 0.0.0.0:3000->3000/tcp``
9) COPY EC2 Public IP paste on browser Access the app insert the data 
10) Output 
11) Check in Your Database
 <img width="1680" height="1050" alt="Screenshot 2025-10-04 at 11 12 57 PM" src="https://github.com/user-attachments/assets/16255b07-981f-4045-88da-f09bb39c738a" />

