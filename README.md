# AWS Incident-Response Demo Lab (Single Account, Free-Tier Friendly)

## 🆓 How to Create a Free AWS Account

This lab is designed to run entirely within the **AWS Free Tier**. To follow along, you’ll need your own personal AWS account.

### Step 1 — Sign up
1. Go to [aws.amazon.com/free](https://aws.amazon.com/free/).
2. Click **Create a Free Account**.
3. Enter your email, choose a password, and set an AWS account name.
4. Enter your contact details (choose **Personal account**).
5. Add a valid credit/debit card (AWS requires this even for free accounts — charges only apply if you exceed the free tier).
6. Identity verification: enter a phone number, receive a call/SMS, and type in the code.
7. Choose a support plan: **Basic (Free)**.
8. Done — your account will be ready in a few minutes.

### Step 2 — Log in
- Go to [console.aws.amazon.com](https://console.aws.amazon.com/).
- Log in as the **root user** (your email) for initial setup.  
- Later, it’s recommended to create an **IAM user** for daily use, but for this demo, root access is enough.

---

## ⚠️ Things to Keep in Mind (Free Tier Caveats)

- **Duration**: The Free Tier is valid for **12 months** from the day you create your AWS account, not just 1 month.
- **Always Free**: Some services (like Lambda, SSM Parameter Store, SNS basic tier) are free even after 12 months.
- **Usage limits**: Free Tier has quotas — if you go beyond them, you will be charged.  
  For example:
  - EC2: 750 hours/month of **t2.micro/t3.micro** (Linux/Windows) for the first 12 months.
  - S3: 5 GB standard storage, 20,000 GET, 2,000 PUT requests.
  - Lambda: 1 million requests and 400,000 GB-seconds compute per month.
  - CloudWatch: 10 custom metrics, 5GB log ingestion.
- **Region matters**: Free tier applies in certain regions (most major ones). Stick to the region you select when creating resources.
- **Billing alerts**:  
  - Set up the **AWS Free Tier Usage Alerts** in the [Billing console](https://console.aws.amazon.com/billing/home).
  - Create a **Budget** with an email alert so you’re notified if charges exceed $0.
- **Don’t leave things running**: If you’re done with the lab, delete the CloudFormation stack — this cleans up EC2, S3 buckets, and Lambdas.

---

## 🛡️ Safety checklist before running the lab
- ✅ Use a **new personal account** (not company AWS account, unless approved).  
- ✅ Confirm you’re still within the **12-month free tier window**.  
- ✅ Subscribe to billing alerts.  
- ✅ Delete the stack after the demo.  

---


This lab sets up a hands-on incident-response environment you can use for a **show-and-tell** with your security team. It is designed for a **single AWS account** and sticks to services with generous **free tiers**.

---

## What the stack builds

- **Networking & Compute**
  - 1 VPC, 1 public subnet, Internet Gateway, route table.
  - 1× EC2 (Amazon Linux 2023) managed via **SSM**.
  - Security Group allowing HTTP (0.0.0.0/0) and SSH from your IP.
- **Storage**
  - **S3 App bucket** (you will try to make public during the demo).
  - **S3 Logs bucket** to store remediation audit records.
  - Block Public Access ON.
- **Observability & Eventing**
  - **CloudTrail Mgmt events → EventBridge** (no paid trail required).
  - **SNS Alerts** (email) for all detections/remediations.
  - **CloudWatch Logs** for Lambda and SSM outputs.
- **Automation**
  - **EventBridge Rules** for: S3 bucket policy/ACL changes, EC2 Security Group wide ingress, SSM Patch compliance changes, and two scheduled checks.
  - **Lambda Functions** for:
    - `S3PublicRevertFn`: Auto-locks down public S3 bucket changes.
    - `SgGuardFn`: Auto-revokes wide (0.0.0.0/0) ingress on sensitive ports.
    - `PatchNowFn`: Triggers `AWS-RunPatchBaseline` when patch compliance is NON_COMPLIANT.
    - `ParamRotatorFn`: Rotates **SSM Parameter Store** SecureString values on a schedule.
    - `IamStaleKeyCleanerFn`: Disables stale IAM access keys (and optionally reissues, storing new keys in Parameter Store).
- **SSM Patch Association**
  - Periodic **Scan** to generate/refresh patch compliance signals.

---

## Architecture Diagram (Mermaid)

> GitHub, VS Code, and many Markdown tools render Mermaid directly.

```mermaid
flowchart TD
  subgraph AWS["AWS Account (Single)"]
    subgraph Net["VPC + Public Subnet"]
      EC2[EC2 (Amazon Linux 2023)\nSSM Agent\nHTTP Server]
      SG[Security Group]
      IGW[Internet Gateway]
      EC2 --- SG
      SG --- IGW
    end

    subgraph Storage["S3 Buckets"]
      APP[(App Bucket)]
      LOGS[(Logs Bucket)]
    end

    subgraph Events["Observability & Eventing"]
      CT[CloudTrail Mgmt Events]
      EB[Amazon EventBridge]
      SNS[SNS Topic: SecurityAlerts]
      CWL[CloudWatch Logs]
    end

    subgraph Autom["Automation"]
      L1[[Lambda: S3PublicRevertFn]]
      L2[[Lambda: SgGuardFn]]
      L3[[Lambda: PatchNowFn]]
      L4[[Lambda: ParamRotatorFn]]
      L5[[Lambda: IamStaleKeyCleanerFn]]
      SSM[SSM Patch/Automation]
    end
  end

  CT --> EB
  EB -- S3 PutBucketPolicy/PutBucketAcl --> L1
  EB -- EC2 AuthorizeSecurityGroupIngress --> L2
  EB -- SSM Compliance Change --> L3
  EB -- rate(...) --> L4
  EB -- rate(...) --> L5

  L1 --> SNS
  L2 --> SNS
  L3 --> SNS
  L4 --> SNS
  L5 --> SNS

  L1 -- audit json --> LOGS

  L3 --> SSM
  SSM --> EC2

  EC2 --> APP
```

---

## Parameters (when launching the CloudFormation stack)

- `AlertEmail`: Email address to receive alerts (**confirm the SNS subscription**).
- `YourIpCidr`: CIDR allowed for SSH to EC2 (e.g., `203.0.113.10/32`).
- `KeyName` (optional): EC2 Key Pair for SSH.
- `InstanceType`: default `t3.micro` (or `t2.micro` if available).
- `ScheduleMinutes`: frequency for the two scheduled checkers (e.g., `5` for demo, later `1440` daily).
- `StaleKeyDays`: age threshold for IAM key staleness (e.g., `90`; for demo, `0`).
- `ParamRotationPrefix`: Parameter Store path to include in rotation (default `/lab/secret/`).
- `ParamRotateAfterDays`: age threshold for rotating parameters (e.g., `30`; for demo, `0`).
- `AutoCreateNewIamKey`: set to `true` to create + store new keys after disabling stale ones.

---

## Deployment Steps

1. **Create Stack**
   - Console → **CloudFormation** → **Create stack** → **With new resources**.
   - Paste the provided YAML template.
   - Set parameters (see above).
   - Create stack and wait for **CREATE_COMPLETE**.

2. **Confirm SNS Subscription**
   - Check inbox for “AWS Notification - Subscription Confirmation” and click **Confirm subscription**.

3. **(Optional) Seed a “secret”**
   - Systems Manager → **Parameter Store** → **Create parameter**
     - Name: `/lab/secret/db-password`
     - Type: **SecureString**
     - Value: any string

---

## Live Demo Script (5 Scenarios)

> Keep **CloudWatch → Logs** open (Lambda log groups) and your **email** for alerts.

### A) Auto-Patch on Non-Compliance (SSM)
1. Systems Manager → Patch Manager → find the instance and **Scan** now.
2. If **NON_COMPLIANT**, EventBridge triggers `PatchNowFn` which runs `AWS-RunPatchBaseline` (Install).
3. **Watch** for email alert “Patch Auto-Remediation” and compliance flipping to **COMPLIANT**.

**Tip:** Temporarily hold a package back or run a version lock to force non-compliance.

---

### B) S3 Bucket Exposure → Auto-Lockdown
1. Open the **App bucket** from the stack outputs.
2. Make it public (add a wildcard policy or public ACL grant).
3. **Watch** for “S3 Public Access Auto-Remediation” email; bucket policy/ACLs are reverted and an **audit JSON** is written to the Logs bucket.

---

### C) Over-Permissive Security Group → Auto-Revoke
1. Edit `ir-demo-ec2-sg` inbound rules and add `SSH` from `0.0.0.0/0`.
2. **Watch** for “Security Group Auto-Remediation” email; wide rule is removed within seconds.

---

### D) Parameter Store “Secrets” → Auto-Rotation
1. Ensure a SecureString param exists under the prefix (e.g., `/lab/secret/app-token`).
2. For a quick demo, set `ParamRotateAfterDays=0` and `ScheduleMinutes=5` (at stack creation).
3. **Watch** for “Parameter Rotation” email; parameter version increments and **Last modified** updates.

---

### E) Stale IAM Access Keys → Disable (and optionally Reissue)
1. Create a test IAM user + access key.
2. For demo, set `StaleKeyDays=0` and `ScheduleMinutes=5`.
3. **Watch** for “IAM Key Remediation” email; old key becomes **Inactive**. If `AutoCreateNewIamKey=true`, the new key is stored in **Parameter Store** under `/iam/<username>/...`.

---

## Cleanup

- CloudFormation → **Delete stack**.
- If deletion is blocked by bucket objects, empty the **Logs** bucket first.

---

## Troubleshooting Tips

- **No emails received**: Confirm SNS subscription; check spam folder; ensure `AlertEmail` is correct.
- **Remediation didn’t trigger**: Verify EventBridge rule matched (check **CloudWatch Logs** for the Lambda). Some console actions take a moment to be recorded by CloudTrail.
- **SSM not running**: Instance must have SSM agent (AL2023 includes it) and the `AmazonSSMManagedInstanceCore` role attached.
- **Parameter rotation did nothing**: Ensure parameter name starts with the configured `ParamRotationPrefix` and is older than `ParamRotateAfterDays`.
- **IAM key reissue**: If `AutoCreateNewIamKey=false`, only disable happens; set to `true` to create and store new keys.
