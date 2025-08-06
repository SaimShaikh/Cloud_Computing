# IAM (Identity and Access Management)

## Scenario

Suppose your company has migrated applications to a cloud platform (like AWS, Azure, or GCP). You need to ensure that only authorized users can access sensitive data and resources. As an administrator, you want to:

- Provide staff access to only what they need.
- Audit user activity.
- Manage permissions centrally.

## What is IAM?

**Identity and Access Management (IAM)** is a framework of policies, technologies, and systems that ensure the right individuals (users, services, applications) have appropriate access to technology resources.

In cloud platforms, IAM enables security and control by allowing you to create and manage users, roles, groups, and permissions.

## Why We Use IAM

- **Security:** Prevent unauthorized access to sensitive resources.
- **Centralized Control:** Manage permissions for all users from a single location.
- **Granular Access:** Assign very specific permissions for users or groups (Principle of Least Privilege).
- **Auditability:** Monitor and log who accesses what, when, and how.

## Benefits of IAM

- **Enhanced Security** through access control.
- **Reduced Risk** of data breaches and leaks.
- **Simplified User Management** via roles and policies.
- **Compliance Support** with activity logs and audits.

## Charges

- **Base IAM services** are generally free across major cloud providers like AWS, Azure, and GCP.
- **Advanced features** (e.g., premium security, extended logging, identity federation) may incur charges—check your provider’s pricing documentation.

## How to Create IAM Resources

### 1. Creating an IAM User

- Log in to your cloud provider’s IAM console.
- Navigate to **Users** > **Add User**.
- Enter a username and choose an authentication method (password or access keys).
- Assign permissions either directly or via group memberships.

### 2. What is a Role?

A **role** is a set of permissions that define the actions allowed for users, services, or resources. Roles are used to delegate access without sharing long-term credentials.

#### Examples:

- `ReadOnlyRole`: Grants view-only access.
- `AdminRole`: Grants full administrative privileges.
- `LambdaExecutionRole`: Grants AWS Lambda permission to access logs or other services during execution.

### 3. How to Create and Assign a Role

#### Creating a Role

1. Go to **IAM > Roles > Create Role**.
2. Choose a **trusted entity** (like a user, service, or another account).
3. Attach **policies** that define what the role can do.
4. Name the role and finalize creation.

#### Assigning a Role

- For **Users**:
  - Go to **IAM > Users > [Username] > Add permissions**.
  - Attach policies directly or through a group.
- For **Services** (e.g., EC2 or Lambda):
  - While creating the service, select or create an IAM role to attach.
  - Or, modify an existing service to attach a role under its security settings.

## Example: AWS IAM Role Creation (Step-by-Step)
Sign in to AWS Management Console

Go to Services > IAM > Roles > Create Role

Choose trusted entity (e.g., AWS service > EC2)

Attach permission policies (e.g., AmazonS3ReadOnlyAccess)

Set Role Name (e.g., EC2S3ReadOnlyRole)

Click Create Role

Go to EC2 > Instances > Select Instance > Actions > Security > Modify IAM Role > Attach Role

