# S3 Bucket Policy -- Restrict Access to a Specific IP Address

## 1. Objective

Create an Amazon S3 bucket and configure a bucket policy so that **only
requests originating from one specific public IP address** can access
the bucket and its objects.

In this lab:

-   **S3 Bucket:** `my-test2-denied-access`
-   **Allowed Public IP:** `49.205.128.237`
-   **Test Object:** `test.txt`
-   **Client:** macOS terminal
-   **IAM User:** IAM user with sufficient S3 permissions /
    AdministratorAccess for testing

The lab verifies:

1.  Access from the allowed IP succeeds.
2.  Access from a different public IP is denied.
3.  An IAM user with AdministratorAccess is still affected by an
    explicit `Deny`.
4.  Removing the bucket policy removes this particular IP restriction;
    the object itself is not permanently modified by the policy.

------------------------------------------------------------------------

## 2. Architecture

``` text
                         Internet
                            |
                            |
                 Public IP: 49.205.128.237
                            |
                            v
                     AWS S3 Bucket
              my-test2-denied-access
                            |
                     Bucket Policy
                            |
             +--------------+--------------+
             |                             |
     49.205.128.237                   Other IP
             |                             |
          ALLOW                         DENY
             |                             |
             v                             X
        test.txt                     AccessDenied
```

------------------------------------------------------------------------

## 3. Prerequisites

-   AWS account
-   IAM user with permission to access S3
-   AWS CLI installed and configured
-   macOS Terminal
-   S3 bucket named `my-test2-denied-access`
-   A known public IP address to allow

Check the AWS CLI:

``` bash
aws --version
```

Check the IAM identity currently configured:

``` bash
aws sts get-caller-identity
```

------------------------------------------------------------------------

## 4. Find the Public IP Address

Run:

``` bash
curl https://checkip.amazonaws.com
```

In this lab, the public IP was:

``` text
49.205.128.237
```

This is the IP that should be allowed by the S3 bucket policy.

> Important: AWS evaluates the public source IP visible to AWS. It does
> not use your Mac's private address such as `192.168.x.x` or
> `10.x.x.x`.

------------------------------------------------------------------------

## 5. Create the S3 Bucket

If the bucket does not already exist:

``` bash
aws s3 mb s3://my-test2-denied-access --region ap-south-1
```

Verify:

``` bash
aws s3 ls
```

Or:

``` bash
aws s3 ls s3://my-test2-denied-access
```

------------------------------------------------------------------------

## 6. Create a Test Object

Create a local test file:

``` bash
echo "IP restriction test" > test.txt
```

Upload it:

``` bash
aws s3 cp test.txt s3://my-test2-denied-access/
```

Verify:

``` bash
aws s3 ls s3://my-test2-denied-access/
```

Expected object:

``` text
test.txt
```

------------------------------------------------------------------------

## 7. Configure the S3 Bucket Policy

Open:

**AWS Console → S3 → `my-test2-denied-access` → Permissions → Bucket
policy**

Use the following policy:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllExceptSpecificIP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-test2-denied-access",
        "arn:aws:s3:::my-test2-denied-access/*"
      ],
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": "49.205.128.237/32"
        }
      }
    }
  ]
}
```

Save the policy.

------------------------------------------------------------------------

## 8. Understand the Policy

The important section is:

``` json
"NotIpAddress": {
  "aws:SourceIp": "49.205.128.237/32"
}
```

The logic is:

``` text
Source IP = 49.205.128.237
        |
        v
NotIpAddress condition = FALSE
        |
        v
Explicit Deny does NOT apply
        |
        v
Continue authorization evaluation
```

For any other IP:

``` text
Source IP != 49.205.128.237
        |
        v
NotIpAddress condition = TRUE
        |
        v
Explicit Deny applies
        |
        v
AccessDenied
```

### Why `/32`?

`49.205.128.237/32` represents exactly one IPv4 address.

``` text
49.205.128.237/32
        |
        +-- Only this IP
```

------------------------------------------------------------------------

## 9. Test Access from the Allowed IP

First confirm the current public IP:

``` bash
curl https://checkip.amazonaws.com
```

Expected:

``` text
49.205.128.237
```

Test bucket access:

``` bash
aws s3 ls s3://my-test2-denied-access
```

Download the object:

``` bash
aws s3 cp s3://my-test2-denied-access/test.txt ~/Desktop/test.txt
```

If successful, verify the downloaded file:

``` bash
cat ~/Desktop/test.txt
```

Expected:

``` text
IP restriction test
```
<img width="2018" height="419" alt="image" src="https://github.com/user-attachments/assets/d3a7e43a-0306-4959-8d36-9aafafa58d15" />

### Result

``` text
49.205.128.237
        |
        v
S3 Bucket
        |
        v
Access allowed
```

------------------------------------------------------------------------

## 10. Test Access from a Different IP

Switch the Mac to another network.

Examples:

-   Mobile hotspot
-   Different Wi-Fi
-   VPN with a different exit IP

Check the new public IP:

``` bash
curl https://checkip.amazonaws.com
```

For example:

``` text
106.x.x.x
```

The exact address will depend on the second network.

Now attempt to download the object:

``` bash
aws s3 cp s3://my-test2-denied-access/test.txt ~/Desktop/test2.txt
```

Expected result:

``` text
An error occurred (AccessDenied) when calling the GetObject operation:
Access Denied
```

This demonstrates that the IP restriction is working.
<img width="2628" height="391" alt="image" src="https://github.com/user-attachments/assets/f3dbf7d8-f15e-4fda-948d-19f3837b59b0" />

------------------------------------------------------------------------

## 11. Why AdministratorAccess Does Not Bypass the Restriction

Suppose the IAM user has:

``` text
AdministratorAccess
```

AdministratorAccess provides broad permissions through an IAM identity
policy.

However, the S3 bucket policy contains an **explicit Deny**.

AWS authorization can be represented as:

``` text
IAM User
    |
    | AdministratorAccess
    |       ALLOW
    v
S3 Request
    |
    v
Bucket Policy
    |
    | Source IP != 49.205.128.237
    |       EXPLICIT DENY
    v
ACCESS DENIED
```

The key rule is:

> An explicit `Deny` overrides an `Allow`.

Therefore:

``` text
AdministratorAccess + Wrong IP
            |
            v
       AccessDenied
```

While:

``` text
AdministratorAccess + Allowed IP
            |
            v
       Access allowed
```

The IAM identity must still have the required permissions when there is
no applicable explicit deny.

------------------------------------------------------------------------

## 12. Bucket Policy vs IAM Policy

### IAM Policy

An IAM policy is an **identity-based policy**.

It answers:

> What is this IAM user, group, or role allowed to do?

Example:

``` json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-test2-denied-access/*"
}
```

### S3 Bucket Policy

A bucket policy is a **resource-based policy** attached to the S3
bucket.

It answers questions such as:

-   Who can access this bucket?
-   From which IP?
-   Through which VPC endpoint?
-   From which AWS account?
-   Under what conditions?

For this lab, the bucket policy controls the source IP.

------------------------------------------------------------------------

## 13. Important Authorization Concept

Think of authorization as:

``` text
                  S3 Request
                      |
          +-----------+-----------+
          |                       |
          v                       v
    IAM Permissions         Bucket Policy
          |                       |
       ALLOW?                Explicit DENY?
          |                       |
          +-----------+-----------+
                      |
                      v
               Final Decision
```

A simplified rule is:

``` text
Explicit Deny
     |
     v
   DENY
```

Otherwise, an applicable Allow must exist.

------------------------------------------------------------------------

## 14. What Happens if the Bucket Policy Is Removed?

The IP restriction is **not permanently attached to the object**.

For example:

``` text
S3 Bucket
└── test.txt
```

The object remains unchanged.

If the bucket policy is removed:

``` text
IP restriction
      |
      v
   REMOVED
```

AWS evaluates the remaining authorization controls.

If the IAM user has permission such as:

``` json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-test2-denied-access/*"
}
```

then the user may be able to download the object from another IP.

Therefore:

> A bucket policy is evaluated during access requests. It does not
> permanently modify the object.

Other controls may still affect access, such as:

-   IAM policies
-   KMS key policies and permissions
-   VPC endpoint policies
-   Service Control Policies (SCPs)
-   S3 Block Public Access settings
-   Other explicit denies

------------------------------------------------------------------------

## 15. Important Caveat: Dynamic Public IP

The allowed IP in this lab is:

``` text
49.205.128.237
```

If your ISP changes your public IP, your access can stop working.

For example:

``` text
Today:
49.205.128.237  → Allowed

Later:
49.205.150.20   → Denied
```

This is because the bucket policy still allows only:

``` text
49.205.128.237/32
```

For production environments, a stable egress IP such as a controlled NAT
Gateway or other fixed egress architecture may be more appropriate.

------------------------------------------------------------------------

## 16. Troubleshooting

### Check current public IP

``` bash
curl https://checkip.amazonaws.com
```

### Check AWS identity

``` bash
aws sts get-caller-identity
```

### Check bucket access

``` bash
aws s3 ls s3://my-test2-denied-access
```

### Download an object

``` bash
aws s3 cp s3://my-test2-denied-access/test.txt .
```

### If access is denied

Check:

1.  Current public IP.
2.  Bucket policy.
3.  IAM permissions.
4.  Whether a VPN is changing the source IP.
5.  Whether the Mac is using another network.
6.  KMS permissions if the object uses SSE-KMS.
7.  Other AWS policies containing explicit Deny.

------------------------------------------------------------------------

## 17. Final Test Matrix

  -----------------------------------------------------------------------
  Test                    Source IP               Expected Result
  ----------------------- ----------------------- -----------------------
  Upload from allowed     `49.205.128.237`        Allowed
  network                                         

  List bucket from        `49.205.128.237`        Allowed if IAM permits
  allowed network                                 

  Download object from    `49.205.128.237`        Allowed
  allowed network                                 

  Download from different Other IP                `AccessDenied`
  network                                         

  IAM user has            Other IP                `AccessDenied`
  AdministratorAccess +                           
  wrong IP                                        

  Remove bucket policy +  Other IP                Allowed, subject to
  IAM GetObject                                   other controls
  permission                                      
  -----------------------------------------------------------------------

------------------------------------------------------------------------
