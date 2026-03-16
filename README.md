# 🔐 Cloud Incident Response + IAM Lockdown

**Automated threat detection and IAM containment using AWS CloudTrail, CloudWatch, SNS, and CLI-driven policy enforcement.**

> A cloud-native incident response pipeline that detects unauthorized privilege escalation in real time and executes automated IAM lockdown — no manual triage required.

---

## 🧩 Problem

Privilege escalation is one of the most common and damaging attack patterns in cloud environments. An attacker — or a compromised internal account — who gains the ability to create or modify IAM policies can effectively own an AWS environment within minutes.

The challenge: most cloud teams detect these events *after the damage is done*, through manual log reviews or delayed SIEM alerts. The goal of this project was to build a detection-to-response pipeline that operates automatically, from the moment a suspicious IAM event fires to the moment access is restricted — without human intervention in the containment step.

---

## 🛠️ Approach

### Architecture
```
CloudTrail → CloudWatch Logs → Metric Filter → CloudWatch Alarm → SNS → IAM Lockdown (CLI)
```

| Component | Role |
|---|---|
| AWS CloudTrail | Captures all API activity including IAM events |
| CloudWatch Logs + Metric Filter | Parses logs for `CreatePolicyVersion` and recon patterns |
| CloudWatch Alarm | Triggers when filtered metric threshold is crossed |
| Amazon SNS | Delivers email alert and signals containment |
| IAM Policy (CLI) | Applies deny policy to restrict compromised user |
| EventBridge | Routes IAM change events to target log group |

### Attack Scenario Simulated

A `suspect-user` account is used to simulate an insider threat or compromised credential. The scenario runs in two phases:

**Phase 1 — Privilege Escalation**
- Suspect account is granted AdministratorAccess via `attach-user-policy`
- New access key is generated, simulating credential theft
- AWS CLI profile is configured using the stolen key
- Attacker performs recon: queries log groups, enumerates IAM structure

**Phase 2 — Detection and Response**
- CloudTrail logs the `CreatePolicyVersion` event from the suspect profile
- Metric filter flags the event; CloudWatch alarm breaches threshold
- SNS sends email alert to security team
- IAM deny policy is applied via CLI to lock out the suspect user

### Key Technical Decisions

- **Metric filter over EventBridge alone:** EventBridge provides near-real-time routing but does not natively support threshold-based alarms. CloudWatch metric filters allow alarm logic (e.g., "trigger if this event appears more than once in 5 minutes") which is essential for avoiding false positives on one-off API calls.
- **CLI-driven lockdown over Lambda automation:** For this lab, IAM lockdown was executed manually via CLI to demonstrate the analyst workflow step-by-step. The architecture is Lambda-ready — the SNS topic can trigger a Lambda function to apply the deny policy automatically, which is the production extension of this pattern.
- **Scoped deny policy:** The lockdown policy uses an explicit deny on `iam:*` and `sts:AssumeRole` rather than a blanket account-wide deny, to avoid disrupting legitimate CloudWatch agent and EC2 service calls during containment.

---

## ✅ Outcome

| Milestone | Result |
|---|---|
| CloudTrail initialized and logging | ✅ Confirmed |
| Root login alarm configured and triggered | ✅ Email alert received |
| Suspect user privilege escalation simulated | ✅ AdministratorAccess attached, access key created |
| Recon activity detected via metric filter | ✅ SNS alarm fired |
| `CreatePolicyVersion` escalation detected | ✅ Alarm threshold crossed |
| IAM lockdown policy applied | ✅ Access restricted via CLI |

The full detection-to-alert pipeline operated end-to-end. A `CreatePolicyVersion` event by the suspect profile triggered the CloudWatch alarm within the metric evaluation window and delivered an SNS email notification. The IAM deny policy was then applied, restricting the compromised account's access to IAM and role assumption.

---

## 📸 Walkthrough

### Infrastructure Setup
![CloudTrail Created](screenshots/01-cloudtrail-creation.png)
*CloudTrail initialized for API activity logging*

![CloudTrail Logging Enabled](screenshots/03-enable-logging.png)
*Logging enabled across all regions for full governance coverage*

![EventBridge Rule Created](screenshots/04-eventbridge-rule-created.png)
*EventBridge rule routes IAM change events to CloudWatch log group*

### Detection Pipeline
![CloudWatch Metric Filter](screenshots/08-cloudwatch-metric-check.png)
*Metric filter targeting `CreatePolicyVersion` events in CloudTrail log group*

![Log Group Filter Fix](screenshots/11-cloudwatch-metric-filter-loggroup-fix.png)
*Corrected filter destination — initial misconfiguration pointed to wrong log group; fixed by verifying log group ARN*

![Root Login Alarm](screenshots/15-cloudwatch-root-login-alarm-configured.png)
*CloudWatch alarm monitoring root account logins — triggers on any root API activity*

![Email Alert Received](screenshots/16-root-login-alarm-email.png)
*SNS email notification delivered on alarm breach*

![CloudTrail Root Event](screenshots/18-cloudtrail-root-events.png)
*Root login event captured and visible in CloudTrail event history*

### Attack Simulation
![Suspect User Exists](screenshots/19-iam-user-already-exists-suspect-user.png)
*Suspect IAM user confirmed in account*

![Admin Policy Attached](screenshots/22-iam-attach-admin-policy-to-suspect-user.png)
*AdministratorAccess policy attached — privilege escalation executed*

![Access Key Created](screenshots/24-access-key-created-suspect-user-aws-compromise.png)
*New access key generated for suspect user — credential exfiltration simulated*

![Profile Configured](screenshots/suspect-configure-profile.PNG)
*AWS CLI profile configured with stolen credentials*

![Recon Activity](screenshots/28-identity-log-groups-output.png)
*Attacker enumerates log groups via CLI — recon behavior logged*

### Response
![Recon Filter Applied](screenshots/29-put-metric-filter-recon.png)
*Metric filter added to detect log group enumeration pattern*

![Alarm Confirmed](screenshots/31-cloudwatch-alarm-confirmation..png)
*CloudWatch alarm created for recon detection threshold*

![SNS Alarm Triggered](screenshots/33-sns-recon-alarm.png)
*Alert delivered when recon threshold breached*

![Escalation Filter](screenshots/34-filter-escalation.png)
*Second metric filter added for `CreatePolicyVersion` escalation detection*

![Lockdown Policy JSON](screenshots/36-policy-json..png)
*Custom deny policy drafted targeting IAM and STS actions*

![Policy Created](screenshots/38-create-policy-v1.png)
*Lockdown policy successfully deployed to account*

![Escalation Alarm Triggered](screenshots/39-create-policy-v2-alarm-trigger.png)
*Alarm fires on `CreatePolicyVersion` event from suspect profile*

![Threshold Crossed](screenshots/40-alarm-threshold-crossed.PNG)
*CloudWatch confirms alarm breach — incident severity confirmed*

---

## 📚 Lessons Learned

**Log group targeting matters more than the filter itself.** The metric filter was correctly written but initially pointed at the wrong log group — CloudTrail was logging to one group while the filter was watching another. The alarm never fired until I verified the ARN explicitly. In production, this would be a silent failure — detection configured but not operational. Lesson: always validate end-to-end with a test event before declaring a detection rule live.

**Threshold tuning is a real tradeoff.** Setting the alarm threshold to 1 (any single occurrence triggers) is appropriate for high-confidence events like `CreatePolicyVersion` by a non-admin account. For noisier events like log group enumeration, a threshold of 1 would produce false positives in a real environment. Distinguishing signal from noise is the core skill in detection engineering.

**CLI lockdown is fast but Lambda is the production pattern.** Manually applying the deny policy via CLI works in a lab and demonstrates the analyst workflow clearly. In production, the SNS topic should trigger a Lambda function that applies the lockdown automatically within seconds of the alarm — removing human latency from the containment step entirely.

---

## 🔧 Tools & Services

- AWS CloudTrail
- AWS CloudWatch Logs & Alarms
- Amazon SNS
- Amazon EventBridge
- AWS IAM
- AWS CLI
- Ubuntu 24.04 (VirtualBox)

---

## 🚀 Planned Extensions

- **Lambda automation:** Replace manual CLI lockdown with SNS-triggered Lambda for sub-60-second automated containment
- **Terraform provisioning:** Infrastructure-as-code for repeatable deployment of the full detection pipeline
- **Additional detection rules:** Expand metric filters to cover `AttachUserPolicy`, `CreateUser`, and `AssumeRole` abuse patterns
- **Integration with Cloud Security Alerting System:** Feed IAM events into the PostgreSQL alerting pipeline for cross-system correlation

---

## ⚠️ Legal Disclaimer

This project was conducted entirely within a personal AWS account using simulated test identities. No unauthorized access, credential use, or interaction with external systems occurred. All content is for educational and professional development purposes only.
