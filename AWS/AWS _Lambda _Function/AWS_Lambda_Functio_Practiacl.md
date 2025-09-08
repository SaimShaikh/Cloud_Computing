
---
Step 1 Go to Lambda function Click On Create Function
<img width="3309" height="1941" alt="image" src="https://github.com/user-attachments/assets/c191f2ef-163f-4fd4-8218-1ae27f0ce8ef" />

---

Step 2 Fill the information 
<img width="3225" height="459" alt="image" src="https://github.com/user-attachments/assets/89d38bc8-0ffe-4d21-adf7-3cc6879f798c" />

---
<img width="3305" height="1798" alt="image" src="https://github.com/user-attachments/assets/8973f854-0344-4b04-83de-ddc908b7bd79" />

---
Step 3 Now You are Lambda Dashboard
<img width="3307" height="1915" alt="image" src="https://github.com/user-attachments/assets/4b16758c-cc3b-46dc-9f4a-d1ef0ce4932b" />

---
Step 4 Go to Test Create Event
<img width="3307" height="1915" alt="image" src="https://github.com/user-attachments/assets/7bb22bee-e56c-4c53-b3a8-7fa92558f62d" />

---
<img width="3272" height="1444" alt="image" src="https://github.com/user-attachments/assets/0920a221-f314-4c65-b8ad-eec1a51b3378" />

Step 5 Go to JSONPlaceholder (https://jsonplaceholder.typicode.com/)
<img width="3190" height="1921" alt="image" src="https://github.com/user-attachments/assets/9343958f-0fc3-454b-b825-db597edd7bca" />

Step 6 Create Config.py and Paste this
`base_url = "https://jsonplaceholder.typicode.com/" `

Step 7 Update New Code and Click on delpoy
```bash
import json 
from config import base_url
import requests

def lambda_handler( event, context):
     res = requests.get(url=base_url+'todos/1')
     return {
              'statusCode': res. status_code,
               'body': res.json()
}
```
<img width="3164" height="915" alt="image" src="https://github.com/user-attachments/assets/392498dd-6b8a-4669-9880-89e6eb564cd5" />

---
### Create or Download ( [requests-layer.zip](https://github.com/user-attachments/files/22216772/requests-layer.zip) ) requests zip if you don’t have and upload in Step 8 
```bash
mkdir -p python <or any name you want>
pip install --target ./python requests
zip -r requests-layer.zip python
```
---
Step 8 Go to layers Create new layer and Add 
<img width="3323" height="934" alt="image" src="https://github.com/user-attachments/assets/4c707f81-cf7b-4809-b662-5ac1b7e50942" />

---

<img width="3320" height="1844" alt="image" src="https://github.com/user-attachments/assets/03353af9-49ab-4afa-9562-aa2ca87fd3d8" />

---

<img width="3221" height="381" alt="image" src="https://github.com/user-attachments/assets/e2b7cd39-687b-45c9-a3b2-d5fc8f983aa3" />

---
Step 9 add and Go on lambda and click on Deploy and Then Test 
<img width="3326" height="1367" alt="image" src="https://github.com/user-attachments/assets/71b504c3-d719-488d-aa12-229480228f31" />

---

OutPut
<img width="3230" height="1411" alt="image" src="https://github.com/user-attachments/assets/fd06539b-4fd5-4deb-941b-5f82989d9cea" />

---

