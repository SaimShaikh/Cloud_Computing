<img width="1280" height="640" alt="image" src="https://github.com/user-attachments/assets/24a37f0e-ac8b-4821-a09e-f325d81cbb9d" />

# Security Groups vs Network Access Control Lists (NACL) in AWS

Understanding the differences between **Security Groups** and **NACLs** is essential for setting up secure networking in AWS.

---

## 🔐 Security Groups

**Definition**: Virtual firewalls that control inbound and outbound traffic at the instance level.

### ✅ Key Characteristics:
- **Stateful**: Return traffic is automatically allowed.
- **Applies to**: EC2 instances.
- **Rules**: Only allow rules (no deny).
- **Evaluation**: All rules are evaluated together (OR logic).
- **Default behavior**:
  - Inbound: All traffic denied.
  - Outbound: All traffic allowed.

### 📌 Example Use Case:
Allow SSH (port 22) and HTTP (port 80) to a web server.

---

## 🛡️ Network ACLs (NACLs)

**Definition**: Subnet-level firewalls that control inbound and outbound traffic.

### ✅ Key Characteristics:
- **Stateless**: Return traffic must be explicitly allowed.
- **Applies to**: Entire subnets.
- **Rules**: Both allow and deny rules.
- **Evaluation**: Rules evaluated in numbered order (lowest first).
- **Default behavior**:
  - Inbound & Outbound: All traffic denied by default.

### 📌 Example Use Case:
Block access to a specific IP range at the subnet level.

---

## 🆚 Key Differences

| Feature                  | Security Groups                      | NACLs                                 |
|--------------------------|--------------------------------------|----------------------------------------|
| **Level of operation**   | Instance-level                       | Subnet-level                          |
| **Stateful/Stateless**   | Stateful                             | Stateless                             |
| **Rule types**           | Only allow                           | Allow and deny                        |
| **Applies to**           | EC2 instances                        | Entire subnets                        |
| **Rule evaluation**      | All rules applied (OR logic)         | Rules evaluated in order              |
| **Use case**             | Instance access control              | Subnet-wide access control            |

---

## ✅ Best Practices

- Use **Security Groups** for granular instance-level control.
- Use **NACLs** for broader subnet-level restrictions (e.g., IP blacklisting).
- Combine both for layered security.

---

## 📚 References

- [AWS Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
- [AWS Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)


