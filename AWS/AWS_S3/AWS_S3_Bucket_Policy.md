# AWS S3 Policy 
An S3 policy is a JSON-based permission document that defines who can access an S3 resource (bucket or object) and what actions they can perform.

There are two main types of policies related to S3:

Bucket Policy – Attached directly to an S3 bucket.

IAM Policy – Attached to a user, group, or role to control access to S3 resources.

Key Points

- Written in JSON format.
- Defines permissions like GetObject, PutObject, DeleteObject, etc.
- Can restrict access based on conditions such as:
   - IP address ranges
   - Specific AWS accounts
   - Enforcing HTTPS
   - MFA authentication

## Example S3 Policy Snippet

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ci-cd-role"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-devops-artifacts/*"
    }
  ]
}
```

