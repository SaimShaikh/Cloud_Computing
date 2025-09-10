# AWS Lambda Function with S3

---

Step 1 Create S3 bucket
<img width="3314" height="1190" alt="image" src="https://github.com/user-attachments/assets/0859bc4c-6bf0-40f0-a741-e59850b3ff8a" />

---

Step 2 Create lambda Function and Create Event 
```bash
import json

def lambda_handler(event, context):
    # TODO implement
    print("New Object added in Bucket")
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

<img width="1680" height="1050" alt="Screenshot 2025-09-10 at 7 22 46 PM" src="https://github.com/user-attachments/assets/55372cb3-91f8-4e9d-b8dd-d066ed01e324" />

---

Step 3 Click on Add trigger
<img width="3317" height="1336" alt="image" src="https://github.com/user-attachments/assets/f7f1b717-39ec-4087-864b-24a32ce95f8c" />

---

Step 4 Select S3
<img width="3337" height="1255" alt="image" src="https://github.com/user-attachments/assets/ba9c7044-ed46-45ff-99da-91abc274e262" />

---

Step 5 Select your bucket which you created 
<img width="3309" height="1798" alt="image" src="https://github.com/user-attachments/assets/a705393d-c46f-4141-8fdd-1b376f442b18" />

---

<img width="3311" height="1892" alt="image" src="https://github.com/user-attachments/assets/81bfd21c-a607-4629-b185-92ce972d08fb" />


---

Step 6 Add object in bucket 
<img width="3323" height="1317" alt="image" src="https://github.com/user-attachments/assets/cb6aaa9f-06a2-480c-bb9b-eb82fe05dd7f" />

---

Step 7 In Lambda click on Monitor then click on View CloudWatch logs
<img width="3316" height="1926" alt="image" src="https://github.com/user-attachments/assets/5d04d4cb-1f2f-452a-8d3c-0fc9b97b0d70" />


---

Step 9 In Logs and select Log groups and then Select Log streams and Click 
<img width="3354" height="1661" alt="image" src="https://github.com/user-attachments/assets/ef20d224-67bd-46bb-b090-9361cedf1df5" />

---

Step 10 Now You can see the Function is called 
<img width="1674" height="733" alt="Screenshot 2025-09-10 at 7 34 17 PM" src="https://github.com/user-attachments/assets/157bbe20-d6fc-4741-914b-9554d16aa66d" />

---
