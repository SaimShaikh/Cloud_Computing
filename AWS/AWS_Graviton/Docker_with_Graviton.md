# AWS Graviton Hands-on Lab 

## Scenario

Your company has a web application running inside a Docker container on an EC2 **t3.medium (Intel x86)** instance.

Management wants to reduce AWS costs without changing the application.

After analysis, the DevOps team decides to migrate the workload to **AWS Graviton (ARM64)** because it provides better price-performance.

Your task is to perform the migration with **zero code changes** and **minimum downtime**.

---

# Final Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/37bc7673-c7ce-4a66-859f-9802ee9ab4fe" />


---

# Lab Prerequisites

* AWS Account
* EC2 Key Pair
* Docker Hub Account
* Docker Desktop Installed
* AWS CLI Installed (Optional)
* Git Installed
* Ubuntu/Linux Terminal

---

# Part 1 - Create Intel EC2

Go to

```
AWS Console

↓

EC2

↓

Launch Instance
```

---

## Name

```
Intel-Server
```

---

## AMI

Choose

```
Amazon Linux 2023

64-bit (x86)
```

DO NOT select ARM.

---

## Instance Type

```
t3.medium
```

---

## Key Pair

Create

```
graviton-key.pem
```

Download it.

---

## Security Group

Allow

```
SSH 22

HTTP 80

HTTPS 443
```

Launch instance.

---

# Part 2 - Connect to EC2

Mac/Linux

```
chmod 400 graviton-key.pem

ssh -i graviton-key.pem ec2-user@PUBLIC_IP
```

Windows

Use PuTTY.

---

# Part 3 - Verify CPU Architecture

```
uname -m
```

Output

```
x86_64
```

---

More details

```
lscpu
```

Expected

```
Architecture: x86_64

CPU op-mode(s): 32-bit,64-bit

Vendor ID: GenuineIntel
```

---

Check OS

```
cat /etc/os-release
```

---

Kernel

```
uname -r
```

---

# Part 4 - Install Docker

Update packages

```
sudo yum update -y
```

Install Docker

```
sudo yum install docker -y
```

Enable Docker

```
sudo systemctl enable docker
```

Start Docker

```
sudo systemctl start docker
```

Verify

```
sudo systemctl status docker
```

Output

```
active (running)
```

---

Add user

```
sudo usermod -aG docker ec2-user
```

Refresh session

```
newgrp docker
```

Verify

```
docker version
```

---

Check Docker

```
docker info
```

---

# Part 5 - Create Demo Application

Create folder

```
mkdir graviton-demo

cd graviton-demo
```

---

Create HTML

```
nano index.html
```

Paste

```html
<html>

<h1>

AWS Graviton Demo

</h1>

</html>
```

Save

```
CTRL + O

ENTER

CTRL + X
```

---

Create Dockerfile

```
nano Dockerfile
```

Paste

```dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html/index.html
```

Save.

---

# Part 6 - Build Local Image

```
docker build -t graviton-demo:v1 .
```

Verify

```
docker images
```

---

Run

```
docker run -d -p 80:80 graviton-demo:v1
```

Verify

```
docker ps
```

Browser

```
http://PUBLIC_IP
```

Output

```
AWS Graviton Demo
```

---

Stop container

```
docker stop CONTAINER_ID

docker rm CONTAINER_ID
```

---

# Part 7 - Login Docker Hub

```
docker login
```

Enter

```
Username

Password
```

---

Verify

```
cat ~/.docker/config.json
```

---

# Part 8 - Enable Buildx

Check

```
docker buildx ls
```

Create builder

```
docker buildx create --name multi-builder --use
```

Bootstrap

```
docker buildx inspect --bootstrap
```

Verify

```
docker buildx ls
```

Current builder

```
docker buildx inspect
```

---

# Part 9 - Build Multi-Architecture Image

```
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t saimeshaikh/graviton-demo:v1 \
--push .
```

Explanation

```
--platform

Creates multiple architectures

linux/amd64

Intel

linux/arm64

AWS Graviton

--push

Uploads directly to Docker Hub
```

---

# Part 10 - Verify Manifest

```
docker manifest inspect saimeshaikh/graviton-demo:v1
```

You should see

```
linux/amd64

linux/arm64
```

Congratulations.

One tag now supports two CPUs.

---

# Part 11 - Launch AWS Graviton EC2

Launch instance.

Name

```
Graviton-Server
```

AMI

```
Amazon Linux 2023

ARM64
```

Instance

```
t4g.medium
```

Launch.

---

# Part 12 - Connect

```
chmod 400 graviton-key.pem

ssh -i graviton-key.pem ec2-user@PUBLIC_IP
```

---

# Part 13 - Verify ARM

```
uname -m
```

Output

```
aarch64
```

---

CPU

```
lscpu
```

Expected

```
Architecture

AArch64
```

Vendor

```
AWS
```

---

# Part 14 - Install Docker

```
sudo yum update -y

sudo yum install docker -y

sudo systemctl enable docker

sudo systemctl start docker

sudo usermod -aG docker ec2-user

newgrp docker
```

Verify

```
docker version
```

---

# Part 15 - Pull Image

```
docker pull saimeshaikh/graviton-demo:v1
```

Docker automatically checks

```
Host CPU

↓

ARM64

↓

Manifest

↓

Downloads linux/arm64 image
```

No manual selection is needed.

---

# Part 16 - Run

```
docker run -d \
-p 80:80 \
--name graviton-demo \
saimeshaikh/graviton-demo:v1
```

Check

```
docker ps
```

Logs

```
docker logs graviton-demo
```

Open browser

```
http://PUBLIC_IP
```

---

# Part 17 - Verify Which Image Was Pulled

```
docker image inspect saimeshaikh/graviton-demo:v1
```

Check

```
Architecture
```

Output

```
arm64
```

---

# Part 18 - What Happens If Image Supports Only amd64?

Suppose

```
docker build \
-t saimeshaikh/demo:v1 .
```

Push

```
docker push saimeshaikh/demo:v1
```

Run

```
docker run saimeshaikh/demo:v1
```

Output

```
standard_init_linux.go

exec format error
```

Reason

```
Intel binary

↓

Running on ARM CPU

↓

Kernel rejects execution
```

---

# Part 19 - Compare Both Servers

Intel

```
uname -m

x86_64
```

Graviton

```
uname -m

aarch64
```

Intel

```
docker pull saimeshaikh/graviton-demo:v1
```

Downloads

```
amd64
```

Graviton

```
docker pull saimeshaikh/graviton-demo:v1
```

Downloads

```
arm64
```

Same command.

Different image.

Docker decides automatically.

---

# Part 20 - Monitor Resources

CPU

```
top
```

Memory

```
free -m
```

Disk

```
df -h
```

Containers

```
docker ps
```

Statistics

```
docker stats
```

Processes

```
ps -ef
```

---

# Part 21 - Blue-Green Migration

```
                Users
                   │
                   ▼
        Application Load Balancer
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
  t3.medium             t4g.medium
  Intel x86             ARM64
        │                     │
      Docker               Docker
```

Migration plan:

* Deploy to Graviton
* Test functionality
* Route 10% traffic
* Monitor logs
* Route 50% traffic
* Monitor metrics
* Route 100% traffic
* Terminate Intel server

---

# Part 22 - Cleanup

Stop

```
docker stop graviton-demo
```

Remove

```
docker rm graviton-demo
```

Delete image

```
docker rmi saimeshaikh/graviton-demo:v1
```

Terminate both EC2 instances from the AWS Console to avoid ongoing charges.

---

# Common Commands Cheat Sheet

## Docker

```
docker images

docker ps

docker ps -a

docker logs CONTAINER_ID

docker exec -it CONTAINER_ID bash

docker stats

docker inspect CONTAINER_ID

docker stop CONTAINER_ID

docker rm CONTAINER_ID

docker image inspect IMAGE
```

---

## Linux

```
uname -m

lscpu

free -m

top

htop

df -h

cat /etc/os-release

hostnamectl
```

---

## Buildx

```
docker buildx ls

docker buildx inspect

docker buildx create --use

docker buildx rm BUILDER_NAME
```

---

## Docker Hub

```
docker login

docker logout

docker push IMAGE

docker pull IMAGE

docker manifest inspect IMAGE
```

---

# Key Learning

* **`t3.medium` uses x86_64 (Intel/AMD).**
* **`t4g.medium` uses ARM64 (AWS Graviton).**
* **A normal Docker image built for `amd64` will fail on Graviton with `exec format error`.**
* **A multi-architecture image built with `docker buildx --platform linux/amd64,linux/arm64` works on both platforms using the same image tag.**
* **Docker automatically selects the correct image variant based on the host CPU architecture.**
