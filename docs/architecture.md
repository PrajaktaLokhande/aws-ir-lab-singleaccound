# Architecture Diagram

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
