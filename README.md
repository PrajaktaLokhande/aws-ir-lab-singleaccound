# AWS Incident‑Response Lab (Single Account, Free‑Tier Friendly)

A hands‑on lab that detects and **auto‑remediates** common risks in a single AWS account using **free‑tier friendly** services: EventBridge, CloudWatch, SNS, Lambda, SSM, and S3.

> **You’ll simulate 5 incidents** and watch alerts + automatic fixes:
> 1) Public S3 bucket → auto‑lockdown  
> 2) Over‑permissive Security Group → auto‑revoke  
> 3) Patch non‑compliance → auto‑patch now  
> 4) Parameter Store secret rotation  
> 5) Stale IAM access keys → disable (and optionally reissue)

---

## 🗺️ Architecture Diagram

```mermaid
flowchart TD
  subgraph AWS["AWS Account (Single)"]
    subgraph Net["VPC + Public Subnet"]
      EC2[EC2 (Amazon Linux 2023)<br/>SSM Agent<br/>HTTP Server]
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

## 🆓 How to Create a Free AWS Account

1. Go to [aws.amazon.com/free](https://aws.amazon.com/free).  
2. Click **“Create a Free Account”**.  
3. Enter your **email**, choose a **password**, and set an **AWS account name** (this will be your account alias).  
4. Choose **Personal account** (not Business) if this is for lab use.  
5. Enter your **contact information** (name, address, phone number).  
6. Add a **credit/debit card** (mandatory, even though Free Tier is free — AWS uses this for identity verification and any usage beyond free tier).  
7. Verify your identity with a **phone call or SMS** (AWS sends a code, you type it in).  
8. Choose a **Support Plan** → select **Basic (Free)**.  
9. Wait a few minutes — you’ll get an email when your account is ready.  

---

## ⚠️ Free Tier Notes

- **12 months** of Free Tier, starting the day you sign up. After that, normal charges apply.  
- Some services are **Always Free** (e.g., Lambda 1M requests/month, 10 custom CloudWatch metrics, SSM Parameter Store, IAM).  
- Free Tier limits (common ones):  
  - EC2: 750 hrs/month of t2.micro/t3.micro.  
  - S3: 5 GB storage, 20k GET, 2k PUT requests.  
  - Lambda: 1M requests/month.  
  - CloudWatch: 5GB log ingestion/month.  
- Free Tier is **per account**, not per region.  
- ⚡ **Set billing alerts**:  
  - Go to **Billing Console → Budgets → Create budget**.  
  - Create a **Zero-dollar budget** to alert you if any charges occur.  
- Delete resources when done (stacks, EC2, S3 buckets).  

---

## Deploy the stack

1. Open **AWS Console → CloudFormation** → **Create stack** → *With new resources*.
2. **Upload**: `cloudformation/ir-demo-lab.yaml` (from this repo).  
3. **Parameters** (recommended for an interactive demo):
   - **AlertEmail**: your email (confirm subscription later).
   - **YourIpCidr**: your IP in CIDR form, e.g., `203.0.113.55/32` (SSH scope).
   - **InstanceType**: `t3.micro` (or `t2.micro`).
   - **ScheduleMinutes**: `5` (so scheduled checks run frequently during the demo).
   - **ParamRotateAfterDays**: `0` (so rotation happens immediately during demo).
   - **StaleKeyDays**: `0` (so stale‑key remediation runs during demo).
   - **AutoCreateNewIamKey**: `true` (to see key re‑issue stored in Parameter Store).
4. Click **Next** → **Next** → Acknowledge capabilities → **Create stack**.
5. When status is **CREATE_COMPLETE**, go to the **Outputs** tab and note:
   - **AppBucketName**
   - **LogsBucketName**
   - **AlertsTopicArn**
   - **InstancePublicIP**

### Confirm the SNS subscription
Open your inbox and confirm the “AWS Notification – Subscription Confirmation”. No email = no alerts.

---

## Console pointers you’ll use

- **S3** → Buckets → `<AppBucketName>` / `<LogsBucketName>`  
- **EC2** → Instances → `ir-demo-ec2` → **Security** tab → Security groups → `ir-demo-ec2-sg`  
- **CloudWatch** → **Logs** → Log groups starting with `/aws/lambda/` (one per Lambda)  
- **EventBridge** → Rules (to see matches and invocations)  
- **SNS** → Subscriptions (confirm `Confirmed` status)  
- **Systems Manager** → **Run Command** and **Parameter Store**  

---

## 🔎 Scenario 1 — S3 bucket made public → auto‑lockdown

**Goal**: Make the app bucket public, see it get **auto‑locked** (BPA re‑enabled, public policy removed), and receive an alert.

1. Console → **S3** → open **`<AppBucketName>`**.
2. **Permissions** tab → **Block public access (bucket settings)** → **Edit** → **uncheck** “Block all public access” → **Save changes** (type *confirm* if asked).
3. Still on **Permissions** tab → **Bucket policy** → **Edit** → paste this policy (replace the bucket name):
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Sid": "PublicRead",
       "Effect": "Allow",
       "Principal": "*",
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::<AppBucketName>/*"
     }]
   }
   ```
   **Save**.

**What to check**
- **Email**: “**S3 Public Access Auto‑Remediation**”  
- **S3 → Permissions**: “**Block all public access**” is back **ON**; Bucket policy is gone or no longer public.  
- **S3 → `<LogsBucketName>`** → prefix **`s3-remediation/`** → newest JSON file has the audit.  
- **CloudWatch Logs**: `/aws/lambda/<stack>-S3PublicRevertFn-...` contains details of the fix.

---

## 🔐 Scenario 2 — Security Group open to world → auto‑revoke

**Goal**: Add SSH from `0.0.0.0/0`; the rule should be **revoked automatically** and you get an alert.

1. Console → **EC2** → **Instances** → choose `ir-demo-ec2` → **Security** tab → **Security groups** → open `ir-demo-ec2-sg`.
2. **Edit inbound rules** → **Add rule**:  
   - Type: **SSH** (22)  
   - Source: **0.0.0.0/0**  
   - **Save rules**.

**What to check**
- The SSH rule disappears shortly (auto‑revoked).  
- **Email**: “**Security Group Auto‑Remediation**”.  
- **CloudWatch Logs**: `/aws/lambda/<stack>-SgGuardFn-...` shows which rule was removed.  
- **EventBridge → Rules**: “EC2 AuthorizeSecurityGroupIngress” rule shows a recent **invocation**.

---

## 🩹 Scenario 3 — Patch non‑compliance → auto‑patch now

**Goal**: When an instance is reported **NON_COMPLIANT** by SSM Patch Manager, auto‑trigger **AWS‑RunPatchBaseline**.

1. Console → **Systems Manager** → **Patch Manager** → **Compliance**.  
2. Locate instance `ir-demo-ec2`.  
3. Choose **Actions → Scan** to refresh compliance.  
   - If status is **NON_COMPLIANT**, the rule triggers **`PatchNowFn`** which calls **Run Command** to install patches.

**What to check**
- **Systems Manager → Run Command → Command history**: an `AWS‑RunPatchBaseline` command was sent to your instance.  
- **Email**: “**Patch Auto‑Remediation**”.  
- **CloudWatch Logs**: `/aws/lambda/<stack>-PatchNowFn-...` shows the trigger.

> If your instance is already compliant, there may be nothing to patch right now. The scan + auto‑patch will still work later whenever non‑compliance is detected.

---

## 🔁 Scenario 4 — Parameter Store secret rotation

**Goal**: Rotate SecureString parameters under a prefix. For a fast demo we set `ParamRotateAfterDays=0` and `ScheduleMinutes=5` at deploy time.

1. Console → **Systems Manager → Parameter Store** → **Create parameter**:  
   - **Name**: `/lab/secret/demo-token`  
   - **Type**: **SecureString** (KMS key = *alias/aws/ssm* is fine)  
   - **Value**: any string (e.g., `initial123!`).
2. Console → **Lambda** → open **`ParamRotatorFn`** → **Test** (create a basic empty test event and **Invoke**).

**What to check**
- **Email**: “**Parameter Rotation**” listing the rotated parameter names.  
- **Parameter Store**: open `/lab/secret/demo-token` → **History** shows a **new version** with a recent **Last modified** timestamp.  
- **CloudWatch Logs**: `/aws/lambda/<stack>-ParamRotatorFn-...` lists rotated items.

---

## 🔑 Scenario 5 — Stale IAM access keys → disable (optionally reissue)

**Goal**: Detect and disable **stale keys** (we set `StaleKeyDays=0` to treat keys as stale immediately) and optionally create a **new key** stored in Parameter Store.

1. Console → **IAM → Users** → **Create user** (access key only).  
2. **Create access key** for this user; download the CSV (for demo only).  
3. Console → **Lambda** → **`IamStaleKeyCleanerFn`** → **Test** (empty event → **Invoke**).

**What to check**
- **Email**: “**IAM Key Remediation**” with the number of disabled keys.  
- **IAM → Users → (user) → Security credentials**: key is now **Inactive**.  
- If `AutoCreateNewIamKey=true`:  
  - **Systems Manager → Parameter Store**: look for  
    `/iam/<username>/access_key_id` and `/iam/<username>/secret_access_key` (SecureString).  
- **CloudWatch Logs**: `/aws/lambda/<stack>-IamStaleKeyCleanerFn-...` has details.

---

## 🔧 Troubleshooting

- **No SNS emails?**  
  - Check if subscription status is **Confirmed** in SNS console.  
  - Look in spam/junk folder.  
- **S3 stack won’t delete?**  
  - Empty the S3 Logs bucket first, then retry.  
- **Lambdas didn’t fire?**  
  - CloudWatch Logs → check the function log group for errors.  
  - EventBridge → check if rules show recent **matches**.  
- **Patch Manager shows no compliance data?**  
  - Make sure SSM agent is running on the instance (Amazon Linux 2023 has it pre-installed).  
  - Instance must have `AmazonSSMManagedInstanceCore` role attached (the template does this).  
- **IAM key cleanup didn’t run?**  
  - Verify you set `StaleKeyDays=0` during stack deploy for instant demo.  
  - Manually invoke the function with a test event in the Lambda console.  

---

## Cleanup

- **CloudFormation** → select the stack → **Delete**.  
- If deletion is blocked by S3 objects, empty the **Logs** bucket (prefix `s3-remediation/`) and retry.
