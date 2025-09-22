# AWS EC2 with Cloud Formation 

---

Step 1. Go to AWS Cloud Formation and click on Create stack
<img width="3329" height="1873" alt="image" src="https://github.com/user-attachments/assets/3d6af969-4bdc-4a14-be4f-7c1245b1fabe" />

---

Step 2. Add the YAML file 
<img width="3329" height="1873" alt="image" src="https://github.com/user-attachments/assets/393141c2-9334-4335-bd24-cda617188879" />

**File Code **
```bash
Resources:
  SimpleEC2Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      InstanceType: t3.micro
      ImageId:  # AMI ID for your region
      Tags:
        - Key: Name
          Value: MySimpleInstance
```

---

Step 3. View and Click on Create Template 
<img width="3323" height="1722" alt="image" src="https://github.com/user-attachments/assets/d0eefe84-7ff5-4c6a-8f3b-c352bd4bdd18" />
<img width="3324" height="1903" alt="image" src="https://github.com/user-attachments/assets/1c392a0f-ed8c-4adb-8e41-3992654a55bf" />
<img width="3338" height="1803" alt="image" src="https://github.com/user-attachments/assets/5615ea1b-3a3e-4154-9914-1ee2bd8ec73f" />

---

Step 4. Specify the stack details 
<img width="3338" height="1785" alt="image" src="https://github.com/user-attachments/assets/a9ba1247-fdc9-4024-9205-dab93e5c4f0c" />
<img width="3317" height="1790" alt="image" src="https://github.com/user-attachments/assets/091132d7-600b-415b-8a43-8d437313ec16" />
<img width="3293" height="1946" alt="image" src="https://github.com/user-attachments/assets/ad5d0ad3-9f9d-44be-b5e1-37d5e38caa7d" />
**Then click Create**
<img width="3336" height="1697" alt="image" src="https://github.com/user-attachments/assets/0ae860cd-a755-47f2-83e2-1685a7970bae" />


---

Step 5. OutPut 
<img width="3334" height="1439" alt="image" src="https://github.com/user-attachments/assets/c93168f3-03dc-4b3d-880d-8e0897a1b5fb" />

---
> Delete all Resources which you created 
