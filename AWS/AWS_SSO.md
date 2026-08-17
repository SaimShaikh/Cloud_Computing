# Single Sign-On (SSO) — End to End Guide + AWS IAM Identity Center Lab

---

## 1. What is SSO (concept, not AWS-specific)

Single Sign-On lets a user authenticate **once** with one identity provider (IdP) and get access to **multiple applications/services** without logging in separately to each.

**Core idea:** Instead of every app checking a username/password against its own database, apps trust a central IdP to vouch for the user's identity via a signed token.

### Key actors
| Actor | Role |
|---|---|
| **User** | The human trying to log in |
| **IdP (Identity Provider)** | Central authority that authenticates the user (Okta, Azure AD/Entra ID, Google Workspace, AWS IAM Identity Center, Keycloak) |
| **SP (Service Provider)** | The application the user wants to access (AWS Console, Salesforce, Slack, an internal app) |

### The trust flow (conceptual)
```
   User                     IdP                        SP (App)
    |                        |                             |
    |--- 1. Try to access -------------------------------->|
    |<-- 2. Redirect to IdP for auth -----------------------|
    |--- 3. Login (user/pass + MFA) ----------------------->|
    |<-- 4. IdP issues signed token/assertion ---------------|
    |--- 5. Present token to SP --------------------------->|
    |<-- 6. SP validates token, grants access ---------------|
```

### Protocols that implement SSO
| Protocol | Token type | Common use |
|---|---|---|
| **SAML 2.0** | XML assertion | Enterprise apps, AWS Console federation, legacy SaaS |
| **OIDC (OpenID Connect)** | JWT (JSON Web Token) | Modern apps, mobile, CLI tools, AWS SSO device flow |
| **OAuth 2.0** | Access token (not identity by itself) | Authorization (delegated access), often paired with OIDC for identity |
| **Kerberos** | Ticket | On-prem Windows AD environments |

**Important distinction:** OAuth2 is about *authorization* (what you can do), OIDC is about *authentication* (who you are) — OIDC is literally built on top of OAuth2. SAML historically does both in one XML blob.

### Why SSO matters (the pitch you'd give in an interview)
- **One identity, many apps** → no password sprawl
- **Centralized offboarding** → disable one account, access to everything is revoked instantly
- **MFA enforced once** at the IdP, inherited everywhere
- **Better audit trail** → all auth events logged centrally
- **No credentials stored in each app** → smaller attack surface

### SSO patterns
1. **IdP-initiated SSO** — user logs into IdP portal first, clicks a tile (e.g., AWS access portal), gets pushed into the app.
2. **SP-initiated SSO** — user goes straight to the app URL, app redirects them to the IdP to authenticate, then bounces back.

---

## 2. SSO in the AWS World — AWS IAM Identity Center

**Naming note (important, comes up constantly in docs/interviews):** AWS renamed **AWS Single Sign-On (AWS SSO)** to **AWS IAM Identity Center** on **July 26, 2022**. Functionality is the same; only the name/branding changed. You'll still see `sso-admin` in CLI commands and `AWSReservedSSO_*` IAM roles — that legacy naming never went away under the hood.

### The problem it solves
Without it: managing access to 10–50 AWS accounts means creating IAM users in *each* account, rotating keys in *each* account, and manually deleting users in *each* account when someone leaves. That doesn't scale and it's an audit nightmare.

With Identity Center: **one central place** to manage human identities and their access across your entire AWS Organization (all accounts) *and* SAML/OIDC-based business apps (Salesforce, Box, Slack, etc.) — from a single login.

### Architecture (how the pieces fit)
```
                     ┌────────────────────────────────────┐
                     │   AWS Organizations (Management)     │
                     │                                      │
                     │   ┌──────────────────────────────┐   │
                     │   │  AWS IAM Identity Center       │   │
                     │   │  - Identity store (users/groups)│  │
                     │   │  - Permission Sets              │  │
                     │   │  - Account Assignments          │  │
                     │   └───────────┬──────────────────┘   │
                     └───────────────┼───────────────────────┘
                                      │ pushes permission sets as IAM roles
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
      ┌───────────────┐      ┌───────────────┐        ┌───────────────┐
      │ AWS Account A  │      │ AWS Account B  │        │ AWS Account C  │
      │ (Dev)          │      │ (Prod)         │        │ (Security)     │
      │ Role:          │      │ Role:          │        │ Role:          │
      │ AWSReservedSSO_│      │ AWSReservedSSO_│        │ AWSReservedSSO_│
      │ DeveloperAccess│      │ AdminAccess    │        │ ReadOnlyAccess │
      └───────────────┘      └───────────────┘        └───────────────┘
                                      ▲
                                      │ browser / CLI login
                              ┌───────┴────────┐
                              │  End User        │
                              │  via AWS access   │
                              │  portal URL        │
                              └────────────────────┘
```

### Core building blocks

**a) Identity source** — where your users/groups live. Three options:
| Option | When to use |
|---|---|
| **Identity Center directory** (default) | Small teams, no existing directory, AWS manages users natively |
| **External IdP via SAML/SCIM** (Okta, Entra ID/Azure AD, Google Workspace, PingOne) | You already have a corporate IdP — this is the enterprise-standard choice |
| **AWS Managed Microsoft AD / on-prem AD via AD Connector** | Company already runs Active Directory |

**b) Permission Sets** — a *template* for an IAM role. When assigned to a user/group + account, Identity Center automatically provisions an IAM role named `AWSReservedSSO_<PermissionSetName>_<hash>` in that account. This is the AWS equivalent of "what can this person do here."

**c) Account Assignments** — the mapping: **which group** gets **which permission set** in **which account(s)**.

**d) AWS access portal** — the single URL (`https://<your-subdomain>.awsapps.com/start`) users log into once, then see tiles for every account+role combo they're allowed into.

**e) Session duration** — how long the temporary credentials issued last (default 1 hour, configurable up to 12 hours per permission set).

---

## 3. Hands-On Setup — Console Walkthrough

### Prerequisites
- An AWS Organizations **management account** (Identity Center works org-wide; enabling it from a member account only gives account-instance scope, not the full multi-account experience)
- Admin permissions on that management account
- Decide your identity source ahead of time (native directory vs external IdP) — switching later is possible but painful

### Step 1 — Enable IAM Identity Center
1. Log into the **management account** of your AWS Organization
2. Console → search **IAM Identity Center**
3. Click **Enable**
4. Choose the Region to host it (pick your primary operating Region — this becomes the **primary Region** and cannot be changed later without deleting the entire configuration and re-enabling from scratch)
5. AWS auto-creates the instance and gives you an **AWS access portal URL** — customize this subdomain now (Settings → Identity source-adjacent "AWS access portal URL" — rename it to something like `yourcompany.awsapps.com/start`, this is a one-time-easy / later-painful change)

### Step 2 — Choose your identity source
Go to **Settings → Identity source → Change identity source**

- **Keep default (Identity Center directory)** — good for labs/small setups. Create users/groups directly in the console.
- **Connect external IdP (SAML 2.0 + SCIM)** — for real orgs:
  1. Identity Center gives you a **SAML metadata file / URL** and an ACS URL
  2. Go to your IdP (say Okta) → create a new SAML app → upload Identity Center's metadata
  3. Copy the IdP's metadata/certificate back into Identity Center
  4. Enable **SCIM provisioning** so users/groups auto-sync from the IdP into Identity Center (no manual duplication)

### Step 3 — Create Users and Groups (if using native directory)
1. **Users** → **Add user** → fill username, email, name → user gets an email to set password + MFA
2. **Groups** → **Create group** → e.g., `Developers`, `Platform-Admins`, `SecurityAudit`
3. Add users to groups — **always assign access via groups, never individual users**. This is the single biggest best practice: it makes onboarding/offboarding a group-membership change instead of touching every account.

### Step 4 — Create Permission Sets
1. **Permission sets** → **Create permission set**
2. Two types:
   - **Predefined permission set** — maps to an AWS managed policy (e.g., `AdministratorAccess`, `ReadOnlyAccess`, `PowerUserAccess`)
   - **Custom permission set** — attach your own managed/customer-managed policies, inline policy, and/or a permissions boundary
3. Set **session duration** (e.g., 1 hour for prod-admin access, up to 12 hours for routine dev access)
4. Name it clearly: `DeveloperAccess`, `ProdReadOnly`, `SecOpsResponder`, etc.

### Step 5 — Assign Access (the part that actually grants permissions)
1. **AWS accounts** tab → select one or multiple accounts (or an OU) → **Assign users or groups**
2. Pick the **group** (e.g., `Developers`)
3. Pick the **permission set** (e.g., `DeveloperAccess`)
4. Confirm — Identity Center provisions the `AWSReservedSSO_DeveloperAccess_xxxx` IAM role in every selected account automatically. This can take a minute or two to propagate.

### Step 6 — User logs in
1. User goes to the **AWS access portal URL**
2. Authenticates (native directory login, or gets redirected to external IdP if configured)
3. Completes MFA
4. Sees a tile per account they have access to, and per role within that account if multiple
5. Clicks through → lands in the AWS Console with temporary credentials **already scoped to that permission set** — no access keys ever touched

---

## 4. CLI Setup (`aws configure sso`) — what you'll actually use day to day

### One-time SSO session setup
```bash
aws configure sso
```
It interactively asks for:
- SSO start URL → `https://yourcompany.awsapps.com/start`
- SSO Region → the Region where Identity Center is hosted
- Opens a browser, you log in, authorize the CLI
- Then pick the account + permission set (role) you want this profile to use
- Names the profile (e.g., `dev-profile`)

This writes config into `~/.aws/config`:
```ini
[profile dev-profile]
sso_session = my-sso
sso_account_id = 111111111111
sso_role_name = DeveloperAccess
region = ap-south-1
output = json

[sso-session my-sso]
sso_start_url = https://yourcompany.awsapps.com/start
sso_region = ap-south-1
sso_registration_scopes = sso:account:access
```

### Daily use
```bash
aws sso login --profile dev-profile     # opens browser, refreshes token
aws s3 ls --profile dev-profile         # temp creds used automatically
```

Credentials auto-refresh silently within the session window; once the session (default up to 8-12h depending on config) expires, `aws sso login` again.

### Admin-side CLI (provisioning as code — `sso-admin`)
```bash
# Create a permission set
aws sso-admin create-permission-set \
  --instance-arn "$SSO_INSTANCE_ARN" \
  --name "ReadOnlyAccess" \
  --description "Read-only access for auditing" \
  --session-duration "PT8H"

# Attach a managed policy to it
aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$SSO_INSTANCE_ARN" \
  --permission-set-arn "arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx" \
  --managed-policy-arn "arn:aws:iam::aws:policy/ReadOnlyAccess"

# Assign group + permission set to an account (this is where access is actually granted)
aws sso-admin create-account-assignment \
  --instance-arn "$SSO_INSTANCE_ARN" \
  --target-id "111111111111" \
  --target-type AWS_ACCOUNT \
  --permission-set-arn "arn:aws:sso:::permissionSet/ssoins-xxxx/ps-xxxx" \
  --principal-type GROUP \
  --principal-id "<group-uuid>"
```
Get `$SSO_INSTANCE_ARN` via:
```bash
aws sso-admin list-instances
```

**Real-world note:** In mature setups this whole flow (permission sets + assignments) is managed via **Terraform** (`aws_ssoadmin_*` resources) or **CloudFormation StackSets**, not clicked/typed manually — click-ops doesn't scale past a handful of accounts. Worth a follow-up lab if you want it.

---

## 5. Scaling pattern — ABAC instead of exploding Permission Sets

Once you have many teams × many environments, RBAC (one permission set per role) multiplies fast: `Dev-Team1`, `Dev-Team2`, `Prod-Team1`... explosion.

**ABAC (Attribute-Based Access Control)** fixes this: tag users with attributes (e.g., `Department=Platform`, `CostCenter=1234`) in the IdP, and write **one** permission set whose policy uses `aws:PrincipalTag` conditions to scope access dynamically per-user, instead of one permission set per team. Identity Center supports passing these attributes for access control — enable it under **Settings → Attributes for access control**.

---

## 5a. Multi-Region Resilience (2026 update)

Identity Center used to be strictly single-Region: whatever Region you enabled it in became a permanent home. That's still true for the **primary Region** (all admin actions — permission set creation, IdP sync, user/group management — only happen there), but AWS has rolled out **multi-Region replication** through 2026:

- **Feb 2026** — general multi-Region support launched: replicate identities, entitlements, and permission set assignments from your primary Region to additional Regions of your choice.
- **Jul 2026** — extended to instances using the native **Identity Center directory** (previously only available for instances federated to an external IdP like Okta).
- **Aug 2026** — **one-click multi-Region setup** for brand-new organization instances: when enabling Identity Center for the first time you can now pick **single-Region**, **multi-Region**, or **custom** straight away, and AWS auto-creates the required multi-Region KMS key for you instead of you configuring it manually.

**What this buys you:** if your primary Region has an outage, users can still log in and access AWS accounts using entitlements already provisioned in the replica Region — you're not fully locked out during a Regional disruption.

**What it doesn't buy you:** it's not a way to *move* your primary Region. Admin operations (creating permission sets, changing IdP config, syncing users) still must happen in the primary Region only. It's available in **17 enabled-by-default commercial Regions** for org instances, and standard KMS charges apply for the customer-managed multi-Region key (Identity Center itself remains free).

**Practical takeaway for setup:** if you're doing this for a production org today, it's worth choosing the **multi-Region** option at enable-time rather than bolting it on later — it's a one-click choice now instead of the old multi-step manual KMS setup.

---

## 6. Troubleshooting Table

| Symptom | Likely Cause | Fix |
|---|---|---|
| User doesn't see any account tiles after login | No account assignment yet for their group | Check **AWS accounts → assignments** in Identity Center console |
| `AccessDeniedException` on an API call | Permission set lacks that permission | Review attached managed/inline policies on the permission set, confirm correct profile/role in use |
| CLI says token expired mid-script | Non-interactive script hit an expired SSO session | Add `aws sso login --sso-session <name>` before automation runs, or better — use a service role / OIDC federation for automated workflows instead of a human SSO session |
| Assigned role not visible in target account's IAM console | Propagation delay, or assignment didn't actually complete | Wait a minute, then check IAM → Roles for `AWSReservedSSO_*`; re-verify the assignment exists in Identity Center |
| Can't change the Identity Center primary Region | Primary Region is locked once enabled | Must fully delete the Identity Center configuration (destructive — wipes permission sets/assignments) and re-enable in the right Region. Note: this is different from adding a *replica* Region for resilience (see Section 5a) — that doesn't require deleting anything |
| SCIM sync not pulling new IdP users | SCIM token expired/misconfigured, or user not assigned to the app in the IdP | Regenerate SCIM token in Identity Center, re-check IdP-side app assignment |
| Users complain of too-frequent logins | Session duration on permission set set too short | Increase session duration (up to 12h) on that permission set |

---

## 7. Interview Q&A Cheat Sheet

**Q: What's the difference between AWS SSO and AWS IAM Identity Center?**
A: Same service — AWS renamed AWS SSO to IAM Identity Center in July 2022. No functional difference; old CLI commands (`sso-admin`) and IAM role prefix (`AWSReservedSSO_`) still carry the old name.

**Q: How is Identity Center different from a regular IAM user?**
A: IAM users have long-lived credentials scoped to one account. Identity Center issues **temporary, auto-expiring credentials**, centrally managed, valid across **multiple accounts** in an Organization, tied to a real human identity rather than a static access key.

**Q: What's a Permission Set?**
A: A reusable template that Identity Center turns into an actual IAM role (`AWSReservedSSO_*`) in each account it's assigned to. It's the "role definition," decoupled from any single account.

**Q: SAML vs OIDC — when would you use which?**
A: SAML is XML-based, mature, and dominant for enterprise SSO into web apps and AWS Console federation. OIDC is JSON/JWT-based, lighter weight, and preferred for modern apps, mobile, and CLI/device-code flows. Identity Center can use SAML to federate with an external IdP, and OIDC internally for CLI device authorization.

**Q: How do you avoid managing 50 IAM users across 50 accounts?**
A: IAM Identity Center + AWS Organizations — one identity, permission sets pushed out as IAM roles into each account, access assigned at the group level.

**Q: Why assign access via groups instead of individual users?**
A: Offboarding/onboarding becomes a single group-membership change instead of updating N account assignments per person — massively reduces audit risk of stale access.

**Q: What happens under the hood when you assign a permission set to an account?**
A: Identity Center provisions an IAM role in that target account named `AWSReservedSSO_<PermissionSetName>_<hash>`, with the permission set's policies attached, and a trust policy that trusts the Identity Center service to assume it via federated login.

---

## 8. Quick Command Cheat Sheet

```bash
# List Identity Center instances
aws sso-admin list-instances

# List permission sets
aws sso-admin list-permission-sets --instance-arn <arn>

# List account assignments for a permission set
aws sso-admin list-account-assignments \
  --instance-arn <arn> --account-id <id> --permission-set-arn <ps-arn>

# CLI login / logout
aws sso login --profile <profile>
aws sso logout

# See who you are currently authenticated as
aws sts get-caller-identity --profile <profile>
```

---

## 9. Mastery Checklist
- [ ] Explain SSO conceptually (IdP vs SP, token-based trust) without AWS in the picture
- [ ] Explain SAML vs OIDC vs OAuth2 differences clearly
- [ ] Enable IAM Identity Center from an Organizations management account
- [ ] Connect an external IdP (SAML) — at least conceptually walk through metadata exchange
- [ ] Create Users/Groups (or explain SCIM auto-provisioning)
- [ ] Create a custom Permission Set with a scoped policy (not just AdministratorAccess)
- [ ] Assign a group + permission set to multiple accounts
- [ ] Log in via the AWS access portal and via `aws configure sso` CLI flow
- [ ] Explain what `AWSReservedSSO_*` roles are and where they live
- [ ] Explain ABAC vs RBAC permission set scaling
- [ ] Know the troubleshooting table above cold for interviews

---

## 10. Cleanup (if this was a sandbox/lab account)
Delete in this order to avoid dependency errors:
1. Remove **account assignments** (users/groups ↔ permission sets ↔ accounts)
2. Delete **permission sets**
3. Delete **users/groups** (if created in native directory)
4. If you connected an external IdP, remove the SAML app on the IdP side too
5. Only disable IAM Identity Center itself if you're fully done — this is destructive and Region-locked to re-enable
