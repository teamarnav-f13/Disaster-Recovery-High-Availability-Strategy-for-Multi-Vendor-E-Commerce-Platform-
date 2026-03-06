# Backup & Replication Strategy
## ShopSmart Inventory Management System - Disaster Recovery

**Version:** 1.0  
**Date:** March 2026  
**Owner:** Team Lead  
**Review Cycle:** Quarterly

---

## 1. Executive Summary

This document defines the backup and replication strategy for the ShopSmart Inventory Management System, ensuring data durability, business continuity, and compliance with recovery objectives.

**Key Metrics:**
- **RPO (Recovery Point Objective):** < 1 minute
- **RTO (Recovery Time Objective):** < 5 minutes
- **Data Durability:** 99.999999999% (11 9's)
- **Service Availability:** 99.99% SLA target

---

## 2. Data Classification

### **Critical Data (Tier 1)**
**Recovery Priority:** Highest  
**RPO:** < 1 minute  
**Backup Frequency:** Real-time replication

- **Products-table:** Product catalog (1,000+ items)
- **VendorInventory:** Stock levels and SKU data (156+ SKUs)
- **StockTransactions:** Financial audit trail (847+ transactions)

### **Important Data (Tier 2)**
**Recovery Priority:** High  
**RPO:** < 15 minutes  
**Backup Frequency:** Continuous replication

- **S3 Product Images:** Visual assets (100+ images)

### **Supporting Data (Tier 3)**
**Recovery Priority:** Medium  
**RPO:** < 1 hour  
**Backup Frequency:** Daily backups

- **CloudWatch Logs:** Application logs (7-day retention)
- **Configuration Data:** Environment variables, IAM policies

---

## 3. DynamoDB Backup Strategy

### **3.1 Point-in-Time Recovery (PITR)**

**Status:** ✅ Enabled on all tables

**Configuration:**
```
Tables with PITR:
- Products-table
- VendorInventory
- StockTransactions

Retention Period: 35 days
Recovery Granularity: Per-second
Cost: ~$0.20 per GB-month
```

**Recovery Procedure:**
1. AWS Console → DynamoDB → Table → Backups
2. Select "Restore to point-in-time"
3. Choose restore timestamp
4. New table name: `[original]-restored-[timestamp]`
5. Wait 5-15 minutes for restore
6. Verify data integrity
7. Update application configuration
8. Switch traffic to restored table

**RTO:** 10-15 minutes  
**RPO:** 0 seconds (continuous backup)

---

### **3.2 On-Demand Backups**

**Schedule:** Weekly (Sunday 23:00 UTC)

**Naming Convention:**
```
[TableName]-weekly-YYYY-MM-DD
Example: Products-table-weekly-2026-03-02
```

**Retention Policy:**
- Weekly backups: 4 weeks
- Monthly backups: 12 months
- Yearly backups: 7 years (compliance)

**Automation:**
```python
# Lambda function: CreateWeeklyBackups
# Trigger: CloudWatch Events (cron: 0 23 * * 0)

import boto3
from datetime import datetime

dynamodb = boto3.client('dynamodb')
tables = ['Products-table', 'VendorInventory', 'StockTransactions']

def lambda_handler(event, context):
    timestamp = datetime.now().strftime('%Y-%m-%d')
    
    for table in tables:
        backup_name = f"{table}-weekly-{timestamp}"
        dynamodb.create_backup(
            TableName=table,
            BackupName=backup_name
        )
```

**Cost:** ~$0.10 per GB per backup

---

### **3.3 Cross-Region Replication (Global Tables)**

**Status:** ✅ Enabled

**Configuration:**
```
Primary Region: ap-south-1 (Mumbai)
DR Region: ap-southeast-1 (Singapore)

Replication Mode: Active-Active (bidirectional)
Replication Lag: < 2 seconds (measured)
Conflict Resolution: Last Writer Wins (timestamp-based)
```

**Monitoring:**
- CloudWatch Metric: `ReplicationLatency`
- Alarm Threshold: > 10 seconds
- SNS Notification: api-alerts-mumbai

**Measured Performance:**
```
Test Results (5 iterations):
- Average Lag: 1.90 seconds
- Min Lag: 1.78 seconds
- Max Lag: 2.01 seconds
- Success Rate: 100%
```

**Failure Handling:**
- Automatic retry with exponential backoff
- Dead Letter Queue for failed replications
- Manual reconciliation procedures documented

---

## 4. S3 Backup Strategy

### **4.1 Cross-Region Replication (CRR)**

**Status:** ✅ Enabled

**Configuration:**
```
Source: shopsmart-inventory-images (ap-south-1)
Destination: shopsmart-inventory-images-dr (ap-southeast-1)

Replication Rule: All objects
Storage Class: S3 Standard (both regions)
Replication Time: < 15 minutes (99.99% within 1 hour)
```

**Prerequisites:**
- ✅ Versioning enabled on both buckets
- ✅ IAM replication role configured
- ✅ CRR rule created and active

**Monitoring:**
```
CloudWatch Metrics:
- BytesPendingReplication
- OperationsPendingReplication
- ReplicationLatency

Alarm: If pending > 1GB for > 1 hour
```

---

### **4.2 S3 Versioning**

**Status:** ✅ Enabled

**Configuration:**
```
Lifecycle Policy:
- Current versions: Retain indefinitely
- Non-current versions: Delete after 30 days
- Delete markers: Remove after 90 days
```

**Recovery Procedure:**
1. AWS Console → S3 → Bucket → Show versions
2. Find deleted/overwritten object
3. Select previous version
4. Actions → Restore

**Use Case:** Accidental deletion or corruption

---

### **4.3 S3 Backup to Glacier (Optional)**

**Status:** ⏳ Not Implemented (Low Priority)

**Future Enhancement:**
```
Transition Policy:
- Images > 90 days old → S3 Glacier Deep Archive
- Cost Savings: ~95% vs S3 Standard
- Retrieval Time: 12-48 hours
```

---

## 5. Cognito Backup Strategy

**Challenge:** Cognito User Pools are regional and cannot be replicated.

**Mitigation Strategy:**

### **5.1 User Data Export**

**Schedule:** Daily

**Method:**
```bash
aws cognito-idp list-users \
  --user-pool-id [POOL_ID] \
  --region ap-south-1 > users-backup-$(date +%Y%m%d).json

# Upload to S3
aws s3 cp users-backup-*.json s3://dr-backups/cognito/
```

**Restoration:** Manual user recreation (scripted)

**RTO:** 1-2 hours (manual process)

---

### **5.2 Accepted Risk**

**Scenario:** Complete Mumbai region failure including Cognito

**Impact:**
- Existing users: Sessions valid for 1 hour (JWT tokens cached)
- New logins: Fail until Mumbai recovers
- Probability: < 0.01% (Cognito 99.99% SLA)

**Business Decision:** Accepted risk vs. cost/complexity of multi-region auth

**Alternative:** Migrate to Auth0 (multi-region support) - $200+/month

---

## 6. Lambda & API Gateway Backup

### **6.1 Infrastructure as Code (IaC)**

**Status:** ✅ Complete

**Repository:** GitHub - disaster-recovery-terraform

**Backup Method:**
- All Lambda code and configuration in Terraform
- Can recreate entire infrastructure from code
- Version controlled with Git

**Recovery Procedure:**
```bash
cd disaster-recovery-terraform/environments/[region]
terraform init
terraform apply
# RTO: 10-15 minutes
```

---

### **6.2 Lambda Code Backup**

**Method:** S3 bucket versioning on deployment packages

**Location:**
```
s3://lambda-deployments-backup/
  ├── ProductCatalogManager-v1.0.zip
  ├── InventoryManager-v1.0.zip
  └── ...
```

**Retention:** 90 days

---

## 7. Configuration Backup

### **7.1 Parameter Store / Secrets Manager**

**Environment Variables:**
```
SSM Parameter Store (encrypted):
- /prod/inventory/db_region
- /prod/inventory/s3_bucket
- /prod/inventory/sns_topic_arn
```

**Backup:** Daily export to S3

---

### **7.2 IAM Policies**

**Backup Method:**
- Export all IAM roles and policies to JSON
- Store in version-controlled repository
- Daily automated export via Lambda

---

## 8. Monitoring & Alerting

### **8.1 Backup Health Monitoring**

**CloudWatch Alarms:**
```
1. PITR Disabled Alarm
   - Metric: PITRStatus
   - Condition: Status != Enabled
   - Action: Critical SNS alert

2. Replication Lag Alarm
   - Metric: ReplicationLatency
   - Condition: > 10 seconds for 2 minutes
   - Action: Warning SNS alert

3. S3 Replication Failure
   - Metric: OperationsPendingReplication
   - Condition: > 1000 for 1 hour
   - Action: Warning SNS alert
```

---

### **8.2 Backup Verification**

**Monthly Drill:**
1. Restore random table from PITR
2. Verify data integrity
3. Delete restored table
4. Document results

**Quarterly Drill:**
1. Full DR failover test
2. Measure RTO/RPO
3. Update runbooks

---

## 9. Disaster Scenarios & Recovery

### **9.1 Scenario: Accidental Table Deletion**

**Detection:** CloudWatch alarm on table deletion event

**Recovery Steps:**
1. Immediately restore from latest PITR backup
2. RTO: 10-15 minutes
3. RPO: 0 seconds (continuous backup)

**Preventions:**
- Enable deletion protection
- Require MFA for table deletion
- Regular backup verification

---

### **9.2 Scenario: Data Corruption**

**Detection:** Application errors, user reports

**Recovery Steps:**
1. Identify corruption timestamp
2. Restore table to point before corruption (PITR)
3. Export corrupted data for analysis
4. Update application with restored table
5. RTO: 20 minutes, RPO: < 1 minute

---

### **9.3 Scenario: Complete Regional Outage**

**Detection:** Route 53 health check failure

**Recovery Steps:**
1. Automatic DNS failover to Singapore (5 minutes)
2. Singapore API serves traffic
3. DynamoDB Global Tables provide data
4. RTO: 5 minutes, RPO: < 2 seconds

**Manual Steps:** None (fully automated)

---

## 10. Compliance & Audit

### **10.1 Data Retention Requirements**
```
Financial Audit Data (StockTransactions):
- Retention: 7 years
- Format: DynamoDB + quarterly exports to S3 Glacier
- Access: Read-only after 90 days

Customer Data (Products, Inventory):
- Retention: Duration of business relationship + 1 year
- Deletion: Automated after retention period
```

---

### **10.2 Backup Testing Log**

| Date | Test Type | Result | RTO | RPO | Notes |
|------|-----------|--------|-----|-----|-------|
| 2026-03-02 | PITR Restore | ✅ Pass | 4m | 0s | Full table restored |
| 2026-03-02 | DR Failover | ✅ Pass | 4m 30s | 1.9s | Automated failover successful |
| 2026-03-02 | Replication Lag | ✅ Pass | N/A | 1.9s | 5 iterations, all passed |

---

## 11. Cost Summary

| Service | Monthly Cost | Annual Cost |
|---------|--------------|-------------|
| DynamoDB PITR (3 tables) | $6.00 | $72.00 |
| DynamoDB Global Tables | $25.00 | $300.00 |
| S3 CRR | $4.00 | $48.00 |
| On-Demand Backups | $2.00 | $24.00 |
| **Total Backup Costs** | **$37.00** | **$444.00** |

**Cost vs. Risk:** Estimated downtime cost: $10,000/hour  
**ROI:** 270:1 (one 4-minute outage prevented pays for entire year)

---

## 12. Responsibilities

| Role | Responsibility |
|------|----------------|
| **Team Lead** | Strategy oversight, escalations |
| **Member 1 (Infrastructure)** | IaC maintenance, Lambda backups |
| **Member 2 (Data)** | DynamoDB backups, PITR monitoring, S3 replication |
| **Member 3 (Operations)** | Monitoring, testing, runbook updates |

---

## 13. Review & Updates

**Document Review:** Quarterly  
**Last Updated:** March 2, 2026  
**Next Review:** June 1, 2026

**Change Log:**
- 2026-03-02: Initial version
- TBD: Updates after first production deployment

---

**Approved By:**  
Team Lead - [Signature]  
Date: March 2, 2026
