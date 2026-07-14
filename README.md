# AWS Cloud Security Monitoring & Alerting Pipeline

Real-time detection and alerting for unauthorized access to sensitive AWS resources, built and tested against a live attack simulation.

## Overview

Unmonitored access to sensitive cloud resources (secrets, credentials, databases) is one of the most common root causes of cloud data breaches. This project builds and validates a detection pipeline that identifies when a secret in AWS Secrets Manager is accessed and alerts a security team in near real-time then tests two different alerting architectures head-to-head to determine which is faster and more useful for incident response.

**Tools:** AWS CloudTrail · Amazon CloudWatch (Alarms + Metric Filters) · Amazon EventBridge · Amazon SNS · IAM

## Architecture


Two parallel detection flows were built against the same CloudTrail event source, to compare their performance:

| | Flow 1 — CloudWatch Metric Filter + Alarm | Flow 2 — EventBridge Rule |
|---|---|---|
| **Trigger source** | Metric filter on ingested CloudTrail logs | Direct CloudTrail event pattern match |
| **Path** | CloudTrail → CloudWatch Logs → Metric Filter → Alarm → SNS → Email | CloudTrail → EventBridge Rule → SNS → Email |
| **Detection speed** | Slower, depends on log ingestion + metric evaluation period | ~3 minutes faster in testing, reacts directly to the event |
| **Alert content** | Basic alarm state change details | Richer — includes username, account ID, source IP |

#### (screenshot) Cloud Watch Alarm configuration:
<img width="1342" height="587" alt="Screenshot (404)" src="https://github.com/user-attachments/assets/7fda630a-a487-4160-8b3c-194feed6ef3e" />

#### (screenshot) EventBridge Rule:
<img width="1224" height="530" alt="Screenshot (405)" src="https://github.com/user-attachments/assets/684c4976-4b2d-4234-aa2d-9ea78a6dd7fd" />

## Attack Simulation & Detection

A simulated unauthorized user (`VictimUser`) attempted to access a sensitive secret (`Production_Database_Credentials`) via the AWS CLI without the required IAM permissions.

**Evidence chain:**
1. **CloudTrail event log** — captured the `GetSecretValue` API call, resulting `AccessDenied` error code, calling identity (IAM user ARN), source IP, and user agent.
2. **Alert delivery** — both flows fired an SNS email notification when the alarm entered `ALARM` state, including alarm name, threshold crossed, and timestamp.
3. **IAM enforcement ("kill-switch") proof** — a follow-up CLI session showed the same unauthorized user being denied on a *second* action (`s3:ListAllMyBuckets`) due to a permissions boundary, confirming least-privilege controls were actually enforced, not just logged.

#### (screenshot) CloudTrail JSON event:
<img width="1346" height="572" alt="Screenshot (406)" src="https://github.com/user-attachments/assets/9cd8634c-5840-4e04-a40a-e89f40e1edb1" />

#### (screenshot) Flow 1 SNS email alert:
<img width="1231" height="530" alt="Screenshot (407)" src="https://github.com/user-attachments/assets/f7969a3b-c5b1-4223-b620-6b86d57b792b" />
#### (screenshot) Flow 2 SNS email alert:
<img width="1244" height="366" alt="Screenshot (408)" src="https://github.com/user-attachments/assets/278016a4-003d-44b4-b122-32e17c6df65e" />


#### (screenshot) IAM CLI AccessDenied proof:
<img width="1286" height="462" alt="Screenshot (409)" src="https://github.com/user-attachments/assets/1f2657d7-6a95-4b92-a8c0-94b3e70d5ea6" />




## Analysis

**Speed:** EventBridge (Flow 2) detected and alerted faster than the CloudWatch Metric Filter path (Flow 1), because it reacts directly to the CloudTrail event stream rather than waiting on log ingestion and metric evaluation cycles.

**Alert quality:** The EventBridge-driven alert carried more actionable context (username, account ID, source IP) — the details an analyst actually needs to triage in the first 60 seconds.

**Response automation:** For automated containment (e.g., disabling a compromised IAM user's access immediately post-alert), the natural next step is an AWS Lambda function triggered by the same EventBridge rule.

**Compliance alignment:** NIST's continuous monitoring guidance requires ongoing visibility into system activity and prompt detection of anomalous behavior. Both flows satisfy this, but EventBridge does so with materially lower latency.

## Recommendation

For a latency-sensitive environment (e.g., financial services), **EventBridge (Flow 2)** is the better choice for triggering automated containment actions, given its faster reaction time and richer alert payload — both of which matter directly for mean-time-to-respond.

## Skills Demonstrated

- Cloud-native log pipeline design (CloudTrail → CloudWatch/EventBridge → SNS)
- Detection engineering trade-off analysis (log-based vs. event-based detection)
- IAM least-privilege verification and negative testing (confirming denies, not just allows)
- Log/evidence analysis from raw JSON (CloudTrail event structure, error codes, identity fields)
- Mapping technical controls to compliance frameworks (NIST continuous monitoring)

