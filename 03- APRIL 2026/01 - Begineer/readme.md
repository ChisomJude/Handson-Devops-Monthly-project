# BEGINNER: AWS Identity Mastery
## Who controls what in the cloud — and how to get it right

**Track:** Cloud Security / DevOps Foundation
**Level:** Beginner — zero to one year, no prior cloud experience needed

**Duration:** 4 weeks
**Format:** Console first → CLI second. Every concept is tested before moving on.

---

## Before You Start — What Is Identity in the Cloud?

When you log into AWS, the first question AWS asks is:

> **"Who are you, and what are you allowed to do?"**

That question — answered correctly — is what separates a secure cloud account from a breached one. Identity is not a feature you add later. It is the foundation everything else is built on.

This project will teach you identity by doing. You will create real users, assign real permissions, test what works and what does not, break things on purpose, and fix them. Every step is done in the AWS Console first so you can see what you are building, then repeated with the CLI so you understand how to automate it.

By the end of this project you will understand:
- What IAM users, groups, roles, and policies are — and when to use each
- How to apply least privilege (only the access that is needed, nothing more)
- How to set up MFA so a stolen password alone cannot compromise an account
- How to stop using long-lived access keys in CI/CD pipelines
- How to audit your account and catch permission mistakes before they become incidents

---

## How This Project Is Structured

Each week is one topic. Each topic follows the same pattern:

```
Concept explained in plain English
        ↓
Build it in the AWS Console (you can see it)
        ↓
Test it — does it work? does the restriction hold?
        ↓
Repeat the same thing with the CLI (automation)
        ↓
What you learned + what to watch out for
```

Do not skip the testing steps. The tests are where the learning happens.

---

---

# WEEK 1
## What is an IAM User? Create one, sign in as them, test their access.

---

### Concept: The Root User vs IAM Users

When you first created your AWS account, you signed in with an email address and password. That login is called the **root user**. The root user has unlimited access to everything in your account — it can delete all your data, rack up a million-dollar bill, or close the account entirely.

You should never use the root user for day-to-day work. Think of it like the master key to a building — you keep it locked in a safe and only use it in emergencies. For all regular work, you create IAM users.

An **IAM user** is an identity you create inside your AWS account. It has:
- Its own username and password (for Console access)
- Its own access keys (for CLI/API access — introduced later)
- Its own set of permissions — you decide exactly what it can do

The key point: **a new IAM user has zero permissions by default.** They can log in but cannot do anything at all until you explicitly grant them access.

---

### Step 1.1 — Secure your root user first

Before anything else, lock down your root user. Do this once, then put it away.

**In the AWS Console:**

1. Sign in at [https://console.aws.amazon.com](https://console.aws.amazon.com) with your root email and password
2. Click your account name (top right) → **Security credentials**
3. Under **Multi-factor authentication (MFA)** → click **Assign MFA device**
4. Choose **Authenticator app** → follow the steps to link Google Authenticator or Authy
5. Under **Access keys** — if any keys exist, **delete them immediately**. Root access keys should never exist.

You have now made it so that even if someone steals your root password, they cannot get in without your phone.

> **What is MFA?**
> Multi-factor authentication means you need two things to log in: something you know (password) and something you have (your phone). Stolen password alone is not enough.

---

### Step 1.2 — Create your first IAM user in the Console

This user will represent a developer on your team. You will sign in as them in the next step.

**In the AWS Console:**

1. Search for **IAM** in the top search bar → open it
2. In the left sidebar → click **Users**
3. Click **Create user** (top right)
4. Fill in:
   - **User name:** `developer-jane`
   - Tick **Provide user access to the AWS Management Console**
   - Select **I want to create an IAM user**
   - Set a custom password (or auto-generate one)
   - Untick "User must create a new password" for now (easier for testing)
5. Click **Next**
6. On the permissions screen — **do not add any permissions yet**. Click **Next**.
7. Review and click **Create user**
8. On the confirmation screen — **download the CSV** or copy the Console sign-in URL, username, and password. You will need these in the next step.

You have created a user with no permissions. They exist. They can log in. They cannot do anything yet.

---

### Step 1.3 — Sign in as `developer-jane` and experience zero permissions

This step is the most important one in Week 1. You need to feel what "no permissions" means before you can appreciate what permissions do.

**Open a private/incognito browser window** (so you stay logged in as root in your main window).

1. Go to the Console sign-in URL from the CSV you downloaded
   - It looks like: `https://YOUR-ACCOUNT-ID.signin.aws.amazon.com/console`
2. Enter the username `developer-jane` and the password
3. You are now logged in as `developer-jane`
4. Search for **S3** → try to open it

You should see something like:

```
You don't have permissions to list buckets.
```

Or S3 will load but every action will fail with an "Access Denied" error.

5. Try **EC2** → same result
6. Try **IAM** → same result

**This is correct behaviour.** A new user with no permissions cannot do anything. AWS is not broken — it is working exactly as designed.

> **What just happened?**
> AWS checked: Does `developer-jane` have permission to call `s3:ListBuckets`? No policy says yes. The default answer is deny. Access denied.
> This is called **implicit deny** — if there is no explicit allow, the answer is always no.

---

### Step 1.4 — Give `developer-jane` one permission and test it

Now you will add exactly one permission — read-only access to S3 — and immediately verify that it works and that nothing else changed.

**Back in your main browser window (logged in as root or admin):**

1. Go to **IAM** → **Users** → click `developer-jane`
2. Click the **Permissions** tab → **Add permissions** → **Attach policies directly**
3. Search for `AmazonS3ReadOnlyAccess` → tick it → click **Next** → **Add permissions**

**Back in your incognito window (logged in as `developer-jane`):**

4. Refresh the page or navigate back to **S3**
5. You should now be able to see buckets and objects (if you have any)
6. Try to **upload a file** → this should still be denied
7. Try to **delete a bucket** → this should still be denied
8. Go to **EC2** → still denied

**What you just proved:**
- Adding `AmazonS3ReadOnlyAccess` gave `developer-jane` the ability to read S3
- It did not give any other permissions — EC2 is still inaccessible
- Read-only means read-only — upload and delete are still blocked

This is least privilege working. The user has exactly what they need, nothing more.

---

### Step 1.5 — Repeat with the CLI (automation)

Now that you understand what you built, repeat it using the AWS CLI. This is how you will manage identity at scale.

**Install the AWS CLI:**

```bash
# macOS
brew install awscli

# Linux (Ubuntu/Debian)
sudo apt-get install awscli -y

# Verify installation
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x ...
```

**Create a CLI access key for your admin user:**

You need credentials for the CLI to authenticate. For now, create a key for your main admin account.

1. In the Console: **IAM** → **Users** → your admin user → **Security credentials**
2. Under **Access keys** → **Create access key**
3. Select **Command Line Interface (CLI)** → acknowledge the warning → **Next** → **Create**
4. Copy the **Access Key ID** and **Secret Access Key** — you only see the secret once

**Configure the CLI:**

```bash
aws configure
# AWS Access Key ID: paste your key ID
# AWS Secret Access Key: paste your secret key
# Default region name: us-east-1
# Default output format: json
```

**Now create `developer-jane` using the CLI:**

```bash
# Create the user
aws iam create-user --user-name developer-bob

# Create a Console login (password)
aws iam create-login-profile \
  --user-name developer-bob \
  --password "Temp@12345!" \
  --no-password-reset-required

# Attach the same read-only S3 policy
aws iam attach-user-policy \
  --user-name developer-bob \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Verify the user was created and the policy is attached
aws iam list-attached-user-policies --user-name developer-bob
```

**Expected output:**

```json
{
  "AttachedPolicies": [
    {
      "PolicyName": "AmazonS3ReadOnlyAccess",
      "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
    }
  ]
}
```

You just did in four commands what took 10 Console clicks. Same result, repeatable, scriptable.

---

### Week 1 — What You Learned

| Concept | What it means |
|---|---|
| Root user | Unlimited access — lock it with MFA, never use day to day |
| IAM user | An identity inside your account with its own credentials and permissions |
| Implicit deny | No permission = access denied. AWS never allows by default. |
| Least privilege | Give only what is needed. Test that nothing extra was granted. |
| Policy | A document that says what actions are allowed or denied on which resources |

**Common mistake to avoid:** New engineers often attach `AdministratorAccess` to every user to "avoid permission issues". This is how breaches happen. Start with nothing and add only what is tested and needed.

---

---

# WEEK 2
## Groups and Policies — manage permissions at scale, write your own policy

---

### Concept: Why Groups Exist

Imagine you have 20 developers. You added S3 read access to each one individually. Now your manager says all developers also need access to CloudWatch logs. You have to update 20 users one by one.

Groups solve this. A **group** is a collection of users that share the same set of policies. You attach policies to the group, add users to the group, and every user in that group inherits those permissions automatically.

When a user is in multiple groups, they get the combined permissions of all groups.

---

### Concept: AWS Managed Policies vs Your Own Policies

AWS provides hundreds of pre-built policies called **managed policies**. `AmazonS3ReadOnlyAccess` is one of them. They are convenient but often broader than you need — `AmazonS3ReadOnlyAccess` allows read access to every bucket in your account, not just one specific bucket.

A **customer managed policy** is one you write yourself. It can be scoped to exactly the resource you intend — one bucket, one function, one specific action. Writing your own policies is the professional approach.

A policy is a JSON document with this structure:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-specific-bucket/*"
    }
  ]
}
```

Four fields to understand:

- **Effect:** `Allow` or `Deny`. That is it.
- **Action:** What API call is being controlled. `s3:GetObject` means "download a file from S3".
- **Resource:** Which specific thing. `*` means everything. A specific ARN means just that resource.
- **Statement:** You can have multiple of these blocks in one policy.

---

### Step 2.1 — Create groups in the Console

**In the AWS Console → IAM → User groups → Create group:**

Create three groups that reflect real team roles:

**Group 1: `platform-engineers`**
- Group name: `platform-engineers`
- Attach policy: `PowerUserAccess` (can do most things, cannot manage IAM)
- Create group

**Group 2: `developers`**
- Group name: `developers`
- Attach policies: `AmazonS3ReadOnlyAccess`, `CloudWatchReadOnlyAccess`
- Create group

**Group 3: `auditors`**
- Group name: `auditors`
- Attach policy: `ReadOnlyAccess` (view everything, change nothing)
- Create group

---

### Step 2.2 — Add users to groups and test inherited permissions

**Add `developer-jane` to the `developers` group:**

1. IAM → User groups → `developers` → **Users** tab → **Add users**
2. Select `developer-jane` → **Add users**

**Remove the direct policy attachment from `developer-jane`:**

This is important — permissions should come from groups, not individual attachments. Mixing both creates confusion.

1. IAM → Users → `developer-jane` → **Permissions** tab
2. Under "Permissions policies" → find `AmazonS3ReadOnlyAccess` → **Remove**
3. Confirm removal

**Test in the incognito window as `developer-jane`:**

- S3 → can still read (permission now comes from the group, not direct attachment)
- CloudWatch → can now read logs (new from the group)
- EC2 → still denied
- IAM → still denied

The user's access did not change from their perspective — but it now comes from a group, making it manageable at scale.

---

### Step 2.3 — Write your first custom policy

**The scenario:** `developer-jane` needs to upload files to one specific S3 bucket called `project-uploads`. She should not be able to read or delete from it — only upload. And she should have no access to any other bucket.

This is more precise than any AWS managed policy. You need to write it yourself.

**In the Console → IAM → Policies → Create policy:**

1. Click **JSON** tab (not visual editor — learn the JSON)
2. Paste this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UploadToProjectBucketOnly",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::project-uploads/*"
    },
    {
      "Sid": "ListProjectBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::project-uploads"
    }
  ]
}
```

3. Click **Next**
4. Policy name: `ProjectUploadsWriteOnly`
5. Description: "Allows uploading files to project-uploads bucket only. No read, no delete, no other buckets."
6. **Create policy**

**Attach it to `developer-jane` directly (for this test):**

1. IAM → Users → `developer-jane` → **Permissions** → **Add permissions** → **Attach policies directly**
2. Search `ProjectUploadsWriteOnly` → attach it

**Create the bucket first (if it does not exist):**

In S3 → **Create bucket** → name it `project-uploads-yourname-12345` (bucket names must be globally unique) → keep all defaults → Create.

Update your policy's Resource ARN to match your actual bucket name.

---

### Step 2.4 — Test the custom policy

Sign in as `developer-jane` (incognito window) and test all four scenarios:

**Test 1 — Upload to the correct bucket:** Try uploading any file to `project-uploads-yourname-12345`
- Expected result: **Success** ✓

**Test 2 — Download from the same bucket:** Try downloading a file you just uploaded
- Expected result: **Access Denied** ✓ (you have `PutObject` but not `GetObject`)

**Test 3 — Delete from the same bucket:** Try deleting a file
- Expected result: **Access Denied** ✓ (no `DeleteObject` permission)

**Test 4 — Upload to a different S3 bucket:** Try uploading to any other bucket
- Expected result: **Access Denied** ✓ (Resource is scoped to one bucket only)

If all four tests produce the expected result, your custom policy is working correctly. This is how you prove that your permissions do exactly what you intended — no more, no less.

---

### Step 2.5 — Repeat with the CLI

```bash
# Create the three groups
aws iam create-group --group-name platform-engineers
aws iam create-group --group-name developers
aws iam create-group --group-name auditors

# Attach policies to groups
aws iam attach-group-policy \
  --group-name platform-engineers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

aws iam attach-group-policy \
  --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-group-policy \
  --group-name auditors \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Add developer-jane to the developers group
aws iam add-user-to-group \
  --group-name developers \
  --user-name developer-jane

# Create your custom policy from a file
# Save the JSON above as project-uploads-policy.json first
aws iam create-policy \
  --policy-name ProjectUploadsWriteOnly \
  --policy-document file://project-uploads-policy.json \
  --description "Upload-only access to project-uploads bucket"

# Attach it to developer-jane
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam attach-user-policy \
  --user-name developer-jane \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ProjectUploadsWriteOnly

# List everything attached to developer-jane to verify
aws iam list-attached-user-policies --user-name developer-jane
aws iam list-groups-for-user --user-name developer-jane
```

---

### Week 2 — What You Learned

| Concept | What it means |
|---|---|
| Group | A container for users that share the same policies. Manage the group, not each user. |
| Managed policy | Pre-built by AWS. Convenient, but often broader than you need. |
| Customer managed policy | Written by you. Scoped precisely to what is needed. |
| Action | The specific API call being allowed or denied. e.g. `s3:PutObject` |
| Resource | The specific AWS resource the action applies to. Scope this as tightly as possible. |
| SID | A label for a statement. Helps you remember what each block does. |
| Testing permissions | Always test what works AND what is blocked. A policy you cannot break is not tight enough. |

**Common mistake to avoid:** Using `"Resource": "*"` in custom policies. Always specify the exact ARN of the resource. Wildcards in resources are how accidental access happens.

---

---

# WEEK 3
## IAM Roles — the right way to give AWS services an identity

---

### Concept: Why Services Should Not Use Users

Imagine you want a Lambda function to read files from S3. You could create an IAM user, generate an access key, paste the key into the Lambda environment variables, and it would work.

This is wrong. Here is why:

- Access keys are long-lived — they do not expire unless you manually rotate them
- Keys stored in code or environment variables get leaked — in git commits, in logs, in screenshots
- When the key is compromised, you have no idea what it was used for or for how long

**IAM roles solve this entirely.** Instead of a static key, a role gives the Lambda function a temporary credential that:
- Lasts only a few hours
- Rotates automatically — AWS generates a new one before the old one expires
- Is never stored anywhere — AWS injects it into the running environment at runtime

Every AWS service that needs to call another AWS service should use a role. Always.

---

### Concept: Trust Policy — Who Can Use This Role?

A role has two parts:

**Part 1 — Trust policy:** Who is allowed to assume (use) this role?
For a Lambda function the trust policy says: "AWS Lambda is allowed to assume this role."

**Part 2 — Permission policy:** What can this role do once assumed?
This is the same kind of policy you wrote in Week 2.

The trust policy is what makes roles different from users. A user exists permanently. A role is assumed temporarily by whoever the trust policy says is allowed to assume it.

---

### Step 3.1 — Create a role for Lambda in the Console

**Scenario:** You have a Lambda function that needs to read files from one S3 bucket. You will create a role with exactly that permission and attach it to the Lambda.

**In the Console → IAM → Roles → Create role:**

1. **Trusted entity type:** AWS service
2. **Use case:** Lambda → click **Next**

   (This is the trust policy. You just told AWS: "Lambda is allowed to assume this role.")

3. Search for and attach `AmazonS3ReadOnlyAccess` → **Next**

   (For now, use the managed policy. You will tighten this in the test step.)

4. Role name: `lambda-s3-reader-role`
5. Description: "Allows Lambda to read from S3. No other permissions."
6. **Create role**

Click into the role you just created and look at two tabs:

- **Trust relationships** — see the JSON that says Lambda can assume this role
- **Permissions** — see the S3 read policy attached

Read both. Understanding these two halves is the whole concept of IAM roles.

---

### Step 3.2 — Attach the role to a Lambda function and test it

**Create a test Lambda function:**

1. Console → **Lambda** → **Create function**
2. Choose **Author from scratch**
3. Function name: `s3-reader-test`
4. Runtime: Python 3.12
5. Under **Permissions** → **Change default execution role** → **Use an existing role** → select `lambda-s3-reader-role`
6. **Create function**

**Write a test that proves the role is working:**

In the Lambda code editor, replace the default code with:

```python
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')

    # List all buckets the role can see
    response = s3.list_buckets()
    buckets = [b['Name'] for b in response['Buckets']]

    return {
        'statusCode': 200,
        'body': f"Buckets accessible: {buckets}"
    }
```

Click **Deploy** → then **Test** (create a test event with any name, default empty JSON is fine).

**Expected result:** The function returns a list of your S3 buckets. The Lambda function called the S3 API using the role's temporary credentials — no access key anywhere in your code.

**Now test what the role cannot do:**

Replace the code with:

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2', region_name='us-east-1')

    try:
        response = ec2.describe_instances()
        return {'statusCode': 200, 'body': 'EC2 access succeeded (unexpected!)'}
    except Exception as e:
        return {'statusCode': 403, 'body': f"EC2 access correctly denied: {str(e)}"}
```

**Expected result:** Access denied error. The role has S3 read permission only. EC2 is blocked.

---

### Step 3.3 — Tighten the role to one specific bucket

`AmazonS3ReadOnlyAccess` gives read access to every bucket in your account. In production, the Lambda should only access the one bucket it actually needs. Let us fix that.

**In the Console → IAM → Policies → Create policy → JSON tab:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadFromInputBucketOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    },
    {
      "Sid": "WriteLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

Replace `YOUR-BUCKET-NAME` with your actual bucket name.

- Policy name: `LambdaInputBucketReadOnly`
- Create policy

**Update the role:**

1. IAM → Roles → `lambda-s3-reader-role` → **Permissions** tab
2. Detach `AmazonS3ReadOnlyAccess`
3. **Add permissions** → Attach `LambdaInputBucketReadOnly`

**Test again from Lambda:**

Update the Lambda code to try listing a bucket it should not access, and one it should. Confirm the correct one works and the other is denied.

This is the production pattern: roles scoped to one service, one resource, minimum actions.

---

### Step 3.4 — Repeat with the CLI

```bash
# Create the trust policy file
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name lambda-s3-reader-role-v2 \
  --assume-role-policy-document file://lambda-trust-policy.json \
  --description "Lambda role: read from input bucket and write logs only"

# Create the scoped permission policy
cat > lambda-s3-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-policy \
  --policy-name LambdaInputBucketReadOnly \
  --policy-document file://lambda-s3-policy.json

aws iam attach-role-policy \
  --role-name lambda-s3-reader-role-v2 \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/LambdaInputBucketReadOnly

# Verify the role — look at both trust and permissions
aws iam get-role \
  --role-name lambda-s3-reader-role-v2 \
  --query 'Role.AssumeRolePolicyDocument'

aws iam list-attached-role-policies \
  --role-name lambda-s3-reader-role-v2
```

---

### Week 3 — What You Learned

| Concept | What it means |
|---|---|
| IAM Role | A temporary identity assumed by a service or person. No permanent credentials. |
| Trust policy | Defines who is allowed to assume the role. Always check this first. |
| Permission policy | Defines what the role can do once assumed. Same JSON format as Week 2. |
| sts:AssumeRole | The API call that exchanges identity for temporary credentials |
| Temporary credentials | AWS generates and rotates them automatically. They expire. Keys do not. |
| Service role | A role whose trust policy allows an AWS service (Lambda, EC2) to assume it |

**Common mistake to avoid:** Attaching managed policies like `AmazonS3FullAccess` to Lambda roles. Your Lambda does not need to create, delete, or configure S3 — it needs to read from one bucket. Write a scoped policy. Test it.

---

---

# WEEK 4
## No More Stored Keys — GitHub Actions deploys to AWS using OIDC

---

### Concept: The Problem with Keys in CI/CD

When teams set up GitHub Actions to deploy to AWS, the first thing most tutorials tell you is:

1. Create an IAM user
2. Generate an access key
3. Paste `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` into GitHub Secrets

This works. It is also one of the most common ways AWS accounts get compromised. Here is why:

- Keys in GitHub Secrets are long-lived. They stay valid until someone manually rotates them.
- If your repo becomes public by accident, the keys are exposed.
- Keys get printed in CI logs, committed in `.env` files, copy-pasted into Slack.
- When a key leaks, you often do not know for weeks.

**OIDC (OpenID Connect) federation is the fix.** Instead of storing keys:

1. GitHub generates a short-lived cryptographic token when your workflow runs
2. AWS verifies the token came from GitHub and from your specific repository
3. AWS gives GitHub a temporary credential that lasts one hour and then expires forever
4. Nothing is stored. Nothing needs to be rotated. Nothing can be leaked from GitHub Secrets.

---

### Concept: How OIDC Trust Works (Plain English)

Think of it like showing your passport at a border crossing.

- GitHub is your country's passport office. It issues a signed token proving "this request came from workflow X in repo Y belonging to org Z."
- AWS is the border agent. It checks: "is this token signed by GitHub? Does it match the repo I was told to trust? Is it still valid?"
- If both checks pass, AWS issues a temporary visa (credentials). When the job finishes, the visa expires.

No keys. No secrets. No rotation. Just a verified handshake.

---

### Step 4.1 — Register GitHub as a trusted identity provider in AWS

**In the Console → IAM → Identity providers → Add provider:**

1. Provider type: **OpenID Connect**
2. Provider URL: `https://token.actions.githubusercontent.com`
3. Click **Get thumbprint** — AWS fetches GitHub's certificate fingerprint automatically
4. Audience: `sts.amazonaws.com`
5. **Add provider**

You have just told AWS: "I trust tokens issued by GitHub's OIDC service."

Look at the provider that was created — you will see the URL, the thumbprint, and the audience. This is the entry point that allows GitHub to talk to AWS.

---

### Step 4.2 — Create a role that GitHub Actions can assume

**In the Console → IAM → Roles → Create role:**

1. **Trusted entity type:** Web identity
2. **Identity provider:** select the GitHub provider you just created (`token.actions.githubusercontent.com`)
3. **Audience:** `sts.amazonaws.com`
4. **GitHub organization:** your GitHub username or org name
5. **GitHub repository:** the name of your repo (e.g. `my-app`)
6. Click **Next**

Now attach a permission policy. For this test, attach `AmazonS3ReadOnlyAccess` (you can tighten this later for real deploys).

7. Click **Next**
8. Role name: `github-actions-deploy`
9. **Create role**

Click into the role → **Trust relationships** tab → **Edit trust policy**. You will see something like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Read this out loud: "Allow GitHub's OIDC provider to assume this role, but only when the request comes from the `main` branch of `YOUR_ORG/YOUR_REPO`." Every other GitHub repo in the world is blocked by that condition.

Copy the **Role ARN** from the role summary — you need it for the next step.

---

### Step 4.3 — Write a GitHub Actions workflow that uses OIDC

In your GitHub repository, create this file at `.github/workflows/aws-test.yaml`:

```yaml
name: Test AWS OIDC connection

on:
  push:
    branches: [main]

# This permission is required — it tells GitHub to issue an OIDC token
permissions:
  id-token: write
  contents: read

jobs:
  test-aws-access:
    name: Verify AWS identity via OIDC
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Assume AWS role via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_ACCOUNT_ID:role/github-actions-deploy
          aws-region: us-east-1
          # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY here — that is the point

      - name: Confirm who we are
        run: aws sts get-caller-identity

      - name: Test S3 access (should work)
        run: aws s3 ls

      - name: Test EC2 access (should fail)
        run: |
          aws ec2 describe-instances 2>&1 || echo "EC2 access correctly denied"
```

Push this to your `main` branch. Watch the Actions run. In the logs you will see:

- The OIDC token being requested from GitHub
- AWS verifying and issuing temporary credentials
- `sts get-caller-identity` returning the `github-actions-deploy` role ARN
- S3 listing working
- EC2 being denied

**Go to your GitHub repo → Settings → Secrets and variables → Actions.** Notice there are no AWS credentials stored there at all. The workflow worked without them.

---

### Step 4.4 — Tighten the role for a real deployment

Once you are confident the OIDC handshake works, replace `AmazonS3ReadOnlyAccess` with a policy scoped to what your CI/CD actually needs — for example, deploying a Lambda function:

**Console → IAM → Roles → `github-actions-deploy` → Permissions:**

Detach `AmazonS3ReadOnlyAccess` and attach a new custom policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DeployLambdaOnly",
      "Effect": "Allow",
      "Action": [
        "lambda:UpdateFunctionCode",
        "lambda:GetFunction",
        "lambda:PublishVersion"
      ],
      "Resource": "arn:aws:lambda:us-east-1:*:function:myapp-*"
    }
  ]
}
```

Your CI/CD pipeline can now deploy Lambda functions — and nothing else. Not EC2, not RDS, not IAM, not S3. Exactly what it needs.

---

### Week 4 — What You Learned

| Concept | What it means |
|---|---|
| OIDC | A standard that lets external systems (GitHub) prove their identity to AWS without keys |
| Identity provider | An external system AWS is configured to trust as a source of identity |
| `sts:AssumeRoleWithWebIdentity` | The action that exchanges an OIDC token for AWS credentials |
| Conditions in trust policy | Rules that restrict which repos, branches, or orgs can assume the role |
| No stored secrets | OIDC credentials are temporary and auto-expire — nothing to rotate or leak |
| `id-token: write` permission | The GitHub Actions permission that enables OIDC token generation |

**Common mistake to avoid:** Adding `"StringLike"` instead of `"StringEquals"` in the trust policy condition. `StringLike` allows wildcards and could let any branch in your repo assume the role. `StringEquals` locks it to exactly `main`.

---

---

## Project Deliverables

| Week | Deliverable | Proof it works |
|---|---|---|
| 1 | IAM user created, signed in, permissions tested | Console screenshot of "Access Denied" with no permissions, then access after policy attach |
| 2 | Three groups created, custom `ProjectUploadsWriteOnly` policy | All four access scenarios tested: upload works, download denied, delete denied, other bucket denied |
| 3 | Lambda role created, Lambda function using it | Lambda successfully lists one bucket, EC2 call correctly fails |
| 4 | GitHub OIDC provider registered, `github-actions-deploy` role | GitHub Actions workflow runs with no stored secrets, role ARN visible in logs |

---

## IAM in One Page — The Mental Model

```
Who are you?               IAM User (human) or IAM Role (service/CI)
        ↓
What do you want to do?    An Action: s3:GetObject, lambda:UpdateFunctionCode
        ↓
On what?                   A Resource: a specific bucket ARN, function ARN
        ↓
Does any policy Allow it?  Check identity policies, resource policies, boundaries
        ↓
Does any policy Deny it?   An explicit Deny always wins — no exceptions
        ↓
Default answer:            DENY — AWS never allows anything you did not explicitly permit
```

---

## What to Explore Next

Once you are confident with the four weeks above, these are the natural next steps — each one builds directly on what you have learned:

**IAM Identity Center (SSO)** — instead of creating IAM users manually, connect AWS to your Google Workspace or Okta so engineers log in with their company email. The identity management moves from AWS Console to your identity provider.

**AWS Organizations and SCPs** — when you have multiple AWS accounts (dev, staging, prod), Service Control Policies let you set account-level rules that no IAM policy inside the account can override. "No one in the dev account can touch prod resources" becomes enforceable.

**Secrets Manager** — the next step after OIDC: storing database passwords, API keys, and tokens securely with automatic rotation. Pairs directly with the IAM roles you built in Week 3.

**IAM Access Analyzer** — an AWS service that reads your policies and flags anything that grants access to the public or to external accounts. Run it on everything you built in this project.

**Move to Intermediate** — take the roles you built here and apply them inside your GitOps pipeline. Every environment (dev, staging, prod) gets its own IAM role. ArgoCD assumes roles to deploy. GitHub Actions assumes a role per environment. Identity is the thread that runs through everything.
