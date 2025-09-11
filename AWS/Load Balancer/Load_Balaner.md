
# Load Balancer 

---

Step 1. Create 2 ec2 
<img width="3319" height="849" alt="image" src="https://github.com/user-attachments/assets/a94b3a94-47ba-47f1-8853-e4c03d096eb6" />

---
Step 2.  Connect and Install Nginx in both ec2 and add code in index.html
```bash
sudo apt-get insatll nginx -y
cd /var/www/html
vim index.html # Create a index.html file and paste 
```
```bash
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Server 1</title>
  <style>
    /* Reset */
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
      background: linear-gradient(135deg, #1e90ff, #87cefa);
      color: #fff;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
      text-align: center;
      overflow: hidden;
    }

    h1 {
      font-size: 4rem;
      text-shadow: 2px 2px 6px rgba(0,0,0,0.3);
      animation: fadeIn 2s ease-in-out;
    }

    p {
      margin-top: 1rem;
      font-size: 1.2rem;
      opacity: 0.9;
    }

    /* Button styling */
    .btn {
      margin-top: 2rem;
      padding: 12px 24px;
      background-color: #fff;
      color: #1e90ff;
      border: none;
      border-radius: 25px;
      font-size: 1rem;
      cursor: pointer;
      font-weight: bold;
      transition: all 0.3s ease;
    }

    .btn:hover {
      background-color: #1e90ff;
      color: #fff;
      box-shadow: 0 4px 12px rgba(0,0,0,0.2);
    }

    /* Floating animation for background */
    .circle {
      position: absolute;
      border-radius: 50%;
      background: rgba(255,255,255,0.2);
      animation: float 15s infinite;
    }

    .circle:nth-child(1) {
      width: 120px;
      height: 120px;
      top: 20%;
      left: 10%;
      animation-duration: 12s;
    }

    .circle:nth-child(2) {
      width: 200px;
      height: 200px;
      bottom: 15%;
      right: 12%;
      animation-duration: 18s;
    }

    .circle:nth-child(3) {
      width: 80px;
      height: 80px;
      bottom: 30%;
      left: 25%;
      animation-duration: 20s;
    }

    @keyframes float {
      0% { transform: translateY(0) rotate(0deg); }
      50% { transform: translateY(-40px) rotate(180deg); }
      100% { transform: translateY(0) rotate(360deg); }
    }

    @keyframes fadeIn {
      from { opacity: 0; transform: scale(0.9); }
      to { opacity: 1; transform: scale(1); }
    }
  </style>
</head>
<body>
  <!-- Floating background circles -->
  <div class="circle"></div>
  <div class="circle"></div>
  <div class="circle"></div>

  <!-- Main content -->
  <h1>This is Server 1 🚀</h1>
  <p>Welcome! Your server is up and running smoothly.</p>
  <button class="btn" onclick="alert('Server 1 is healthy ✅')">Check Server Status</button>
</body>
</html>
```

* This is Server 2 file 
```bash
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Server 2</title>
  <style>
    /* Reset */
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
      background: linear-gradient(135deg, #1e90ff, #87cefa);
      color: #fff;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
      text-align: center;
      overflow: hidden;
    }

    h1 {
      font-size: 4rem;
      text-shadow: 2px 2px 6px rgba(0,0,0,0.3);
      animation: fadeIn 2s ease-in-out;
    }

    p {
      margin-top: 1rem;
      font-size: 1.2rem;
      opacity: 0.9;
    }

    /* Button styling */
    .btn {
      margin-top: 2rem;
      padding: 12px 24px;
      background-color: #fff;
      color: #1e90ff;
      border: none;
      border-radius: 25px;
      font-size: 1rem;
      cursor: pointer;
      font-weight: bold;
      transition: all 0.3s ease;
    }

    .btn:hover {
      background-color: #1e90ff;
      color: #fff;
      box-shadow: 0 4px 12px rgba(0,0,0,0.2);
    }

    /* Floating animation for background */
    .circle {
      position: absolute;
      border-radius: 50%;
      background: rgba(255,255,255,0.2);
      animation: float 15s infinite;
    }

    .circle:nth-child(1) {
      width: 120px;
      height: 120px;
      top: 20%;
      left: 10%;
      animation-duration: 12s;
    }

    .circle:nth-child(2) {
      width: 200px;
      height: 200px;
      bottom: 15%;
      right: 12%;
      animation-duration: 18s;
    }

    .circle:nth-child(3) {
      width: 80px;
      height: 80px;
      bottom: 30%;
      left: 25%;
      animation-duration: 20s;
    }

    @keyframes float {
      0% { transform: translateY(0) rotate(0deg); }
      50% { transform: translateY(-40px) rotate(180deg); }
      100% { transform: translateY(0) rotate(360deg); }
    }

    @keyframes fadeIn {
      from { opacity: 0; transform: scale(0.9); }
      to { opacity: 1; transform: scale(1); }
    }
  </style>
</head>
<body>
  <!-- Floating background circles -->
  <div class="circle"></div>
  <div class="circle"></div>
  <div class="circle"></div>

  <!-- Main content -->
  <h1>This is Server 2 🚀</h1>
  <p>Welcome! Your server is up and running smoothly.</p>
  <button class="btn" onclick="alert('Server 2 is healthy ✅')">Check Server Status</button>
</body>
</html>
```

---

Step 3 Now Go to Load Balancer and Create load balancer 

<img width="3334" height="967" alt="image" src="https://github.com/user-attachments/assets/ba938092-e9c4-40fb-8c65-68ebc4502117" />

---
Step 4 Select Application Load Balancer and Create 
<img width="2343" height="1821" alt="image" src="https://github.com/user-attachments/assets/2e3436ff-a018-4735-8605-8481cc0e39cd" />

---

Step 5 Fill the from 
<img width="3274" height="1651" alt="image" src="https://github.com/user-attachments/assets/1ab6d9ec-b309-47ba-86e8-99d4fd269fd5" />

---
Step 6 Select all Availability zone 
<img width="3265" height="1541" alt="image" src="https://github.com/user-attachments/assets/d24b3fb9-1e4f-440c-abd2-f5af0648def7" />

---

Step 7 Select Security group which you created 
<img width="3289" height="513" alt="image" src="https://github.com/user-attachments/assets/e164f183-8db1-4170-8789-3b37eeca28ad" />

---
Step 8 Click on Create target group
<img width="3263" height="990" alt="image" src="https://github.com/user-attachments/assets/66a924aa-4ea6-4c97-87f5-4f8b6275be8c" />

---
Step 9 Select Instances in  Specify group details 
<img width="3350" height="974" alt="image" src="https://github.com/user-attachments/assets/e40199e5-5a65-47eb-ac1e-ccba96d49650" />
<img width="3229" height="1384" alt="image" src="https://github.com/user-attachments/assets/664073dc-e2a0-403c-940d-016869ddeecb" />

---

Step 10 Select all Available instances and click Include as pending below
<img width="3351" height="1885" alt="image" src="https://github.com/user-attachments/assets/71df84cb-62b1-4aa6-a124-e76671a3f8e8" />

---

Step 11 Go back into Listeners and routing Info and select target group which you created 
<img width="3196" height="1004" alt="image" src="https://github.com/user-attachments/assets/f8f219a8-9d4d-4d8e-a72a-d1119c3939e6" />

---

Step 12 Now our Load balancer has been created and copy DNS link and paste on web browser 
<img width="3317" height="1949" alt="image" src="https://github.com/user-attachments/assets/8ec96a29-404b-4fcc-8b7b-25ecb899d941" />

### Output 
> Now you are on Server 1 
<img width="3311" height="1973" alt="image" src="https://github.com/user-attachments/assets/a76ac882-afab-450d-a01e-4f4b203ffcd2" />

> Refresh now you are on Server 2
<img width="3255" height="1796" alt="image" src="https://github.com/user-attachments/assets/ee2e7ddd-6d7f-48d7-863a-e005e39c6366" />

---
> After Completing Practical delete all the services 
