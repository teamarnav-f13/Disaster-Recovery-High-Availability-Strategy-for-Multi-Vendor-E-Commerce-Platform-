# Disaster Recovery Operations Runbook
## Vendor Sales & Finance Analytics Dashboard

**Version:** 1.0  
**Last Updated:** March 13, 2026  
**Owner:** Team Lead  
**On-Call Contact:** [Your Contact Info]

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [DR Architecture](#dr-architecture)
3. [Recovery Procedures](#recovery-procedures)
4. [Rollback Procedures](#rollback-procedures)
5. [Health Check Procedures](#health-check-procedures)
6. [Contact Information](#contact-information)
7. [Appendix](#appendix)

---

## 1. System Overview

### 1.1 Application Details

**System Name:** Vendor Sales & Finance Analytics Dashboard  
**Purpose:** Multi-vendor e-commerce sales tracking and financial analytics  
**Business Criticality:** HIGH  
**Users:** 50+ vendors, internal finance team  

### 1.2 Infrastructure Components

| Component | Primary (Mumbai) | DR (Singapore) |
|-----------|------------------|----------------|
| **API Gateway** | `0wek2322jl` | `d9wo0hh9q9` |
| **Lambda Functions** | 4 functions | 4 functions (replicated) |
| **DynamoDB** | VendorSales (Global Table) | VendorSales (replica) |
| **Cognito** | User Pool (Mumbai only) | N/A |
| **Route 53** | Health Check ID: `6b0190c7-f241-466e-b939-a1cf0768f4f0` | Failover Record |

### 1.3 Recovery Objectives

| Metric | Target | Actual (Tested) |
|--------|--------|-----------------|
| **RTO** | < 5 minutes | 5 seconds ✅ |
| **RPO** | < 1 minute | 1.09 seconds ✅ |
| **Availability** | 99.9% | 99.99% (estimated) |

---

## 2. DR Architecture

### 2.1 High-Level Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                      Route 53 DNS                            │
│              api.vendorsales.company.com                     │
│                                                              │
│  Primary (Mumbai)          Failover          DR (Singapore)  │
│  Health Check: ✓                             Standby         │
└─────────────────────────────────────────────────────────────┘
         │                                            │
         ▼                                            ▼
┌──────────────────────┐                  ┌──────────────────────┐
│   Mumbai (ap-south-1) │                  │ Singapore (ap-southeast-1) │
├──────────────────────┤                  ├──────────────────────┤
│ API: 0wek2322jl      │ ◄───Failover───► │ API: d9wo0hh9q9      │
│                      │                  │                      │
│ Lambda Functions (4) │                  │ Lambda Functions (4) │
│ - Transaction        │                  │ - Transaction        │
│ - Orders             │                  │ - Orders             │
│ - Dashboard_Data     │                  │ - Dashboard_Data     │
│ - analytics-alerts   │                  │ - analytics-alerts   │
│                      │                  │                      │
│ DynamoDB Global Table│ ◄───Replication──│ DynamoDB Replica     │
│ VendorSales          │    ~1.1 seconds  │ VendorSales          │
│                      │                  │                      │
│ Cognito User Pool    │                  │ (Uses Mumbai pool)   │
└──────────────────────┘                  └──────────────────────┘
```

### 2.2 Data Flow

**Normal Operations (Primary Active):**
```
User → Route 53 → Mumbai API → Mumbai Lambdas → Mumbai DynamoDB
                                                      ↓
                                        (Replication) ↓
                                                      ↓
                                         Singapore DynamoDB
```

**During Failover (DR Active):**
```
User → Route 53 → Singapore API → Singapore Lambdas → Singapore DynamoDB
```

---

## 3. Recovery Procedures

### 3.1 Automatic Failover (No Manual Intervention)

**Trigger Conditions:**
- Mumbai API Gateway returning 5XX errors for 2+ consecutive health checks (60 seconds)
- Complete Mumbai region outage
- Network partition isolating Mumbai

**Automatic Actions:**
1. Route 53 health check fails (T+30s, T+60s)
2. Route 53 marks Mumbai endpoint unhealthy (T+90s)
3. DNS updated to point to Singapore (T+90s to T+150s)
4. SNS alert sent to on-call team (T+90s)
5. Traffic begins routing to Singapore (T+150s to T+300s)

**Expected Timeline:**
- Detection: 60-90 seconds
- DNS propagation: 60-120 seconds
- Full recovery: 3-5 minutes

**Monitoring During Failover:**
```bash
# Watch Route 53 health check status
aws route53 get-health-check-status \
  --health-check-id 6b0190c7-f241-466e-b939-a1cf0768f4f0 \
  --region us-east-1

# Monitor CloudWatch dashboard
https://console.aws.amazon.com/cloudwatch/home?region=ap-south-1#dashboards:name=VendorSales-DR-Dashboard
```

---

### 3.2 Manual Failover (Forced)

**When to Use:**
- Planned Mumbai region maintenance
- Mumbai API degraded but not failing health checks
- Testing DR procedures

**Procedure:**

#### Step 1: Verify DR Region Health
```bash
# Test Singapore API
curl https://d9wo0hh9q9.execute-api.ap-southeast-1.amazonaws.com/prod/health

# Expected: {"status": "healthy"}
```

#### Step 2: Update Route 53 DNS Record

**AWS Console Method:**
```
1. AWS Console → Route 53 → Hosted Zones
2. Select your domain
3. Find record: api.vendorsales.company.com
4. Edit the Primary (Mumbai) record:
   - Change "Health check" to "None"
   - Save
5. This forces failover to Secondary (Singapore)
```

**AWS CLI Method:**
```bash
# Get current record set
aws route53 list-resource-record-sets \
  --hosted-zone-id YOUR_ZONE_ID \
  --query "ResourceRecordSets[?Name=='api.vendorsales.company.com.']"

# Update to remove health check (forces failover)
# [Use change-resource-record-sets command]
```

#### Step 3: Verify Failover
```bash
# Check DNS resolution
dig api.vendorsales.company.com

# Should resolve to Singapore API Gateway IPs

# Test API
curl https://api.vendorsales.company.com/health

# Should return 200 OK from Singapore
```

#### Step 4: Monitor Application

**CloudWatch Dashboard:**
- Mumbai traffic should drop to zero
- Singapore traffic should increase
- All Lambda invocations should succeed
- DynamoDB requests served from Singapore

**Estimated Time:** 2-3 minutes

---

### 3.3 Lambda-Based Automated Failover Test

**Lambda Function:** `VendorAPI_SimpleFailoverTest`

**Execute via AWS Console:**
```
1. AWS Console → Lambda → VendorAPI_SimpleFailoverTest
2. Test tab → Configure test event:

{
  "mumbaiApiId": "0wek2322jl",
  "singaporeApiId": "d9wo0hh9q9",
  "testEndpoint": "https://0wek2322jl.execute-api.ap-south-1.amazonaws.com/prod/health",
  "executeFailover": true
}

3. Click "Test"
4. Monitor logs for RTO measurement
```

**Expected Results:**
- Phase 1: Pre-flight checks PASS
- Phase 2: Mumbai stage deleted
- Phase 3: Recovery detected in 5-10 seconds
- Phase 4-5: Data consistency verified

---

## 4. Rollback Procedures

### 4.1 Restore Mumbai as Primary

**When to Execute:**
- Mumbai region issues resolved
- Planned maintenance complete
- DR test complete

#### Procedure:

**Step 1: Verify Mumbai API Health**
```bash
# Check API Gateway stage exists
aws apigateway get-stage \
  --rest-api-id 0wek2322jl \
  --stage-name prod \
  --region ap-south-1

# If stage doesn't exist, redeploy:
aws apigateway create-deployment \
  --rest-api-id 0wek2322jl \
  --stage-name prod \
  --region ap-south-1
```

**Step 2: Test Mumbai Endpoint**
```bash
curl https://0wek2322jl.execute-api.ap-south-1.amazonaws.com/prod/health

# Expected: {"status": "healthy"}
```

**Step 3: Re-enable Route 53 Health Check**
```
AWS Console → Route 53 → Health Checks
→ mumbai-vendor-api-health-check
→ Verify status: Healthy ✓

If unhealthy:
- Check endpoint URL
- Verify /prod/health returns 200
- Wait 2-3 minutes for health check to update
```

**Step 4: Update DNS to Restore Primary**
```
AWS Console → Route 53 → Hosted Zones → Your domain
→ api.vendorsales.company.com (Primary record)
→ Edit
→ Re-associate health check: mumbai-vendor-api-health-check
→ Save

Traffic will automatically fail back to Mumbai (primary preference)
```

**Step 5: Verify Failback**
```bash
# Check DNS
dig api.vendorsales.company.com

# Should resolve to Mumbai IPs

# Monitor CloudWatch
# Mumbai traffic should increase
# Singapore traffic should decrease (return to standby)
```

**Estimated Time:** 3-5 minutes

---

### 4.2 Emergency Restore (If Singapore Fails During DR)

**Scenario:** Mumbai failed, now Singapore failing too

**Immediate Actions:**

1. **Page senior engineering leadership**
2. **Post incident alert to #vendor-sales-status Slack channel**
3. **Execute emergency Lambda recovery:**
```bash
# Run emergency restore Lambda
aws lambda invoke \
  --function-name EmergencyAPIRestore \
  --region ap-south-1 \
  --payload '{"action":"force_restore"}' \
  response.json
```

4. **If Lambda fails, manual API Gateway restoration:**
```bash
# Mumbai
aws apigateway create-deployment \
  --rest-api-id 0wek2322jl \
  --stage-name prod \
  --region ap-south-1

# Singapore
aws apigateway create-deployment \
  --rest-api-id d9wo0hh9q9 \
  --stage-name prod \
  --region ap-southeast-1
```

5. **Contact AWS Support** (Enterprise Support, TAM if available)

---

## 5. Health Check Procedures

### 5.1 Daily Health Verification

**Automated (Runs every 4 hours via EventBridge):**
- Lambda: `DailyDRHealthCheck`
- Sends Slack/SNS alerts if any issues

**Manual Verification:**
```bash
#!/bin/bash
# daily-health-check.sh

echo "=== DR Health Check ==="
echo "Date: $(date)"

# Test Mumbai
echo "Mumbai API:"
curl -s https://0wek2322jl.execute-api.ap-south-1.amazonaws.com/prod/health | jq

# Test Singapore
echo "Singapore API:"
curl -s https://d9wo0hh9q9.execute-api.ap-southeast-1.amazonaws.com/prod/health | jq

# Check Route 53
echo "Route 53 Health Check:"
aws route53 get-health-check-status \
  --health-check-id 6b0190c7-f241-466e-b939-a1cf0768f4f0 \
  --region us-east-1 \
  | jq '.HealthCheckObservations[0].StatusReport.Status'

# Check DynamoDB replication
echo "DynamoDB Replication Status:"
aws dynamodb describe-table \
  --table-name VendorSales \
  --region ap-south-1 \
  | jq '.Table.Replicas[] | {Region: .RegionName, Status: .ReplicaStatus}'
```

**Expected Output:**
```
Mumbai API: {"status":"healthy"}
Singapore API: {"status":"healthy"}
Route 53: "Success"
DynamoDB: All replicas "ACTIVE"
```

---

### 5.2 Weekly DR Drill

**Schedule:** Every Monday 2:00 AM UTC

**Procedure:**
1. Announce in #vendor-sales-team: "DR drill starting"
2. Execute `VendorAPI_SimpleFailoverTest` Lambda with `executeFailover: true`
3. Verify RTO < 5 minutes
4. Verify RPO < 60 seconds
5. Restore Mumbai as primary
6. Document results in DR drill log

**Drill Log Location:** `s3://vendor-sales-dr-logs/weekly-drills/`

---

## 6. Contact Information

### 6.1 On-Call Rotation

| Role | Primary | Backup |
|------|---------|--------|
| **Team Lead** | [Name] - [Phone] | [Name] - [Phone] |
| **Member 1 (Infra)** | [Name] - [Phone] | [Name] - [Phone] |
| **Member 2 (Data)** | [Name] - [Phone] | [Name] - [Phone] |
| **Member 3 (Ops)** | [Name] - [Phone] | [Name] - [Phone] |

### 6.2 Escalation Path

1. **L1:** On-call engineer (see rotation above)
2. **L2:** Engineering Manager - [Name] - [Phone]
3. **L3:** CTO - [Name] - [Phone]
4. **External:** AWS TAM (if Enterprise Support) - [Contact]

### 6.3 Communication Channels

- **Slack:** #vendor-sales-incidents
- **PagerDuty:** vendor-sales-oncall
- **Email:** vendor-sales-team@company.com
- **Status Page:** status.vendorsales.company.com

---

## 7. Appendix

### 7.1 Quick Reference Commands

**Check API Health:**
```bash
# Mumbai
curl https://0wek2322jl.execute-api.ap-south-1.amazonaws.com/prod/health

# Singapore
curl https://d9wo0hh9q9.execute-api.ap-southeast-1.amazonaws.com/prod/health
```

**Check Route 53 Health:**
```bash
aws route53 get-health-check-status \
  --health-check-id 6b0190c7-f241-466e-b939-a1cf0768f4f0 \
  --region us-east-1
```

**Check DynamoDB Global Table:**
```bash
aws dynamodb describe-table \
  --table-name VendorSales \
  --region ap-south-1
```

**Force API Gateway Deployment:**
```bash
aws apigateway create-deployment \
  --rest-api-id 0wek2322jl \
  --stage-name prod \
  --region ap-south-1
```

---

### 7.2 Known Issues & Workarounds

**Issue 1:** Cognito tokens invalid after Mumbai outage
- **Impact:** Users must re-login
- **Workaround:** None - expected behavior
- **Duration:** Until Mumbai restored or JWT expires (1 hour)

**Issue 2:** Cold start latency in Singapore after long idle
- **Impact:** First requests may take 2-3 seconds
- **Mitigation:** Lambda warming enabled (pings every 5 minutes)

**Issue 3:** Route 53 DNS caching
- **Impact:** Some users may still hit Mumbai for up to 60 seconds
- **Mitigation:** DNS TTL set to 60 seconds (minimum practical)

---

### 7.3 Change Log

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-03-13 | 1.0 | Initial runbook creation | Team Lead |
| | | | |

---

### 7.4 Review Schedule

- **Monthly:** Review and update contact information
- **Quarterly:** Full runbook review and DR drill
- **Annually:** Comprehensive DR strategy review

---

**Document Status:** ✅ APPROVED  
**Next Review Date:** June 13, 2026  
**Runbook Owner:** Team Lead
