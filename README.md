# ☁️ Cloud Security Alerting System — AWS & PostgreSQL

**Full-stack detection and alerting pipeline using Terraform, AWS RDS PostgreSQL, CloudWatch, and EventBridge — correlating EC2 activity logs with user identity to generate severity-tiered security alerts.**

---

## 🧩 Problem

Cloud security teams generate enormous volumes of event data, but raw logs alone don't produce actionable alerts. The challenge is correlation — linking an event (what happened) to an identity (who did it) and a severity level (how bad is it) in a way that supports fast investigation.

This project simulates that workflow: ingesting EC2 activity events into a relational database, mapping them to user roles, and querying the combined data to surface high-severity alerts the way a SOC analyst would investigate them.

---

## 🛠️ Approach

### Architecture
```
AWS EC2 → EventBridge Rule → CloudWatch Logs → PostgreSQL (AWS RDS) → Alert Queries (psql)
```

| Component | Role |
|---|---|
| AWS EC2 | Event source — state changes trigger detection pipeline |
| Amazon EventBridge | Routes EC2 state change events to CloudWatch |
| CloudWatch Logs | Captures and stores event stream |
| AWS RDS (PostgreSQL) | Relational store for event correlation and alerting |
| Ubuntu 24.04 Jumpbox | Secure query interface via psql |
| Terraform | Infrastructure-as-code provisioning for RDS and EC2 |

### Data Model

Three tables power the alert correlation workflow:
```sql
cloud_event_logs  →  users  →  alerts
     (event_id)        (user_id)     (event_id)
```

- `cloud_event_logs` — EC2 actions with timestamps, event types, and user attribution
- `users` — User roles, departments, and access levels
- `alerts` — Severity classifications tied to specific events

### Key Technical Decisions

- **PostgreSQL over NoSQL:** EC2 event data has well-defined relationships between events, users, and alerts. A relational model with JOIN queries mirrors how real SIEM platforms correlate structured log data — and makes investigation queries readable and auditable.
- **Terraform for provisioning:** Manually clicking through the AWS console doesn't produce repeatable infrastructure. Used Terraform to provision the RDS instance and EC2 jumpbox so the entire environment can be torn down and rebuilt consistently — the same requirement in any production security lab or compliance audit.
- **Jumpbox pattern for RDS access:** Rather than exposing the RDS endpoint publicly, routed all psql queries through an EC2 jumpbox on the same VPC. This mirrors the network segmentation pattern used in production environments where databases are never directly internet-accessible.
- **Severity-tiered alerting:** Classified alerts as Low, Medium, and High based on event type — EC2 terminations from unknown sources flagged as High, routine state changes as Low. Threshold logic mirrors what a SOC analyst would configure in a real alerting rule.

---

## ✅ Outcome

| Milestone | Result |
|---|---|
| Terraform provisions RDS + EC2 successfully | ✅ Confirmed |
| EventBridge rule routes EC2 events to CloudWatch | ✅ Events captured |
| PostgreSQL tables created with relational schema | ✅ All three tables live |
| Seed data inserted and validated | ✅ Events, users, alerts populated |
| JOIN queries return correlated alert output | ✅ User attribution confirmed |
| High-severity termination alert generated | ✅ Flagged and queryable |

Full pipeline operational. EC2 state change events flow through EventBridge into CloudWatch, get stored in RDS, and are queryable via JOIN across all three tables — returning event type, user identity, and alert severity in a single result set.

### Sample Query Output
```sql
SELECT u.username, e.event_type, a.severity
FROM users u
JOIN cloud_event_logs e ON u.user_id = e.user_id
JOIN alerts a ON e.event_id = a.event_id
WHERE a.severity = 'High';
```

Returns: username, event type, and High severity classification for every flagged event — the core investigative query a SOC analyst would run first.

---

## 📸 Walkthrough

### Infrastructure
![EC2 Instance](screenshots/initial-ec2-instance-view.png)
*EC2 instance deployed as PostgreSQL query jumpbox*

![Security Group Review](screenshots/security-group-review.png)
*Security group configured — inbound access scoped to jumpbox only, RDS not publicly exposed*

![RDS Instance Details](screenshots/rds-instance-details.png)
*PostgreSQL RDS instance configuration — Multi-AZ disabled for lab cost, enabled in production pattern*

![Terraform RDS Provision](screenshots/terraform-rds-provision-success.png)
*Terraform apply output confirming successful RDS provisioning*

![Terraform Output Values](screenshots/terraform-output-values.png)
*Terraform outputs: RDS endpoint, VPC ID, and connection parameters*

![Postgres Endpoint](screenshots/postgres-endpoint-connect.png)
*RDS endpoint string used for psql connection from jumpbox*

![Successful psql Connection](screenshots/successful-psql-connection.png)
*Terminal confirmation of authenticated PostgreSQL session from jumpbox*

### Data Model & Seeding
![Tables Created](screenshots/created-custom-tables.png)
*cloud_event_logs, users, and alerts tables confirmed in database*

![Data Inserts](screenshots/data-inserts.png)
*INSERT statements seeding EC2 events, user records, and alert classifications*

![Cloud Logs and User Table](screenshots/cloud-logs-and-user-table.png)
*Side-by-side view of cloud_event_logs and users tables*

![Additional User Insert](screenshots/additional-user-insert.png)
*New user added to test cross-role correlation queries*

![Red Team User](screenshots/red-team-user.png)
*Red Team engineer user record — role attribution for high-severity event*

### Detection Pipeline
![CloudWatch Event Policy](screenshots/cloudwatch-event-policy.png)
*IAM policy enabling CloudWatch event delivery from EventBridge*

![EventBridge Rule](screenshots/eventbridge-rule-preview.png)
*EventBridge rule targeting EC2 state change events*

![CloudWatch Trigger Test](screenshots/cloudwatch-trigger-test.png)
*Manual trigger test to validate CloudWatch rule end-to-end*

![CloudWatch Logs Creation](screenshots/cloudwatch-logs-creation.png)
*Log group created to capture EC2 event stream*

![EC2 Shutdown Log](screenshots/ec2-shutdown-log.png)
*EC2 shutdown event captured in CloudWatch log stream*

![Second Event Log](screenshots/second-event-log.png)
*Second EC2 termination event — multiple events confirm pipeline reliability*

![CloudWatch Dashboard](screenshots/cloudwatch-dashboard-view.png)
*CloudWatch dashboard monitoring EC2 state transitions in real time*

### Alert Correlation
![Full Event Query Output](screenshots/full-event-query-output.png)
*Combined query returning all events with user and severity data*

![High Severity Termination](screenshots/high-severity-termination.png)
*High-severity EC2 termination alert surfaced via JOIN query*

![High Severity Alert Generated](screenshots/high-severity-alert-generated.png)
*Alert record confirmed in alerts table with correct severity classification*

![Joined Data View](screenshots/joined-data-view.png)
*Three-table JOIN confirming event, user, and alert linkage*

![System Query with User Attribution](screenshots/system-query-with-user-attribution.png)
*System-level event attributed to specific user via query*

![Jumpbox psql Output](screenshots/jumpbox-psql-query-output.png)
*Terminal output from jumpbox confirming full query pipeline operational*

![Final Query Validation](screenshots/final-query-validation.png)
*Final JOIN validation — alert, event, and user data confirmed linked correctly*

---

## 📚 Lessons Learned

**Network segmentation is a design decision, not an afterthought.** The jumpbox pattern — routing psql through an EC2 instance rather than exposing RDS directly — added setup complexity but reflects how production databases are actually protected. Building the secure pattern from the start rather than retrofitting it later is the right habit to build.

**Terraform output values are underused.** Initially hard-coded the RDS endpoint into psql commands. Switching to Terraform output values made the connection string dynamic and reusable — if the RDS instance is rebuilt, the endpoint updates automatically. Small change, significant operational improvement.

**Seed data quality determines alert quality.** Early test queries returned misleading results because the seed data didn't have consistent user_id foreign keys across tables. Fixing the data model before writing queries — rather than patching queries to work around bad data — mirrors the discipline required when building detection rules against real log sources.

---

## 🔧 Tools & Technologies

- AWS EC2
- AWS RDS (PostgreSQL)
- Amazon EventBridge
- Amazon CloudWatch
- Terraform
- Ubuntu 24.04 (Jumpbox)
- psql CLI

---

## 🚀 Planned Extensions

- **Lambda integration:** Trigger automated response actions when High severity alerts are inserted into the alerts table
- **Real-time ingestion:** Replace manual seed data with a Lambda function that writes CloudWatch events directly to RDS as they occur
- **Dashboard:** Build a lightweight Grafana or Metabase dashboard on top of the PostgreSQL alerting data
- **Cross-system correlation:** Feed IAM events from the Cloud Incident Response project into the same pipeline for unified multi-source alerting

---

## ⚠️ Legal Disclaimer

This project was conducted entirely within a personal AWS account using simulated test data. No unauthorized access or interaction with external systems occurred. All content is for educational and professional development purposes only.
