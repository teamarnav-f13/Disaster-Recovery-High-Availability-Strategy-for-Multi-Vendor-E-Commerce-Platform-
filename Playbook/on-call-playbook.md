# On-Call Playbook - Disaster Recovery

## Alert Response Procedures

### CRITICAL: API Gateway 5XX Errors

**Alert:** mumbai-api-5xx-errors OR singapore-api-5xx-errors

**Severity:** HIGH

**Response Time:** Immediate (< 5 minutes)

#### Investigation Steps
1. Check CloudWatch Dashboard
   - Which region is affected?
   - Error rate increasing or stable?
   - Which endpoints are failing?

2. Check Lambda metrics
   - Are specific functions erroring?
   - Check Lambda logs in CloudWatch

3. Check DynamoDB
   - Any throttling?
   - Table status: Active?

#### Resolution Steps
**If isolated function errors:**
- Check Lambda logs for error details
- Redeploy function if necessary
- No DR failover needed

**If regional API Gateway failure:**
- Route 53 should auto-failover
- Verify Singapore API responding
- No manual action needed unless failover doesn't occur

**If failover doesn't occur:**
- Manual DNS update in Route 53
- Change primary record to point to Singapore
- Estimated time: 2 minutes + DNS propagation

---

### WARNING: Lambda Function Errors

**Alert:** mumbai-productcatalog-errors (or any Lambda alarm)

**Severity:** MEDIUM

**Response Time:** 15 minutes

#### Investigation Steps
1. Open CloudWatch Logs for affected Lambda
2. Search for error patterns
3. Check recent deployments (was there a recent change?)

#### Resolution Steps
1. If deployment issue: Rollback to previous version
2. If data issue: Check DynamoDB for corrupted data
3. If dependency issue: Check SNS, S3, or external service status

---

### CRITICAL: Health Check Failure

**Alert:** route53-health-mumbai-failed

**Severity:** CRITICAL

**Response Time:** Immediate

#### Investigation Steps
1. Verify if this is a real outage or false positive
   - Try accessing Mumbai API manually
   - Check AWS Service Health Dashboard

2. Check if automatic failover occurred
   - Route 53 → Health Checks → Status
   - CloudWatch Dashboard → Singapore traffic increasing?

#### Resolution Steps
**If real outage:**
- Automatic failover should occur
- Monitor Singapore API taking over traffic
- Investigate Mumbai outage cause
- Document incident

**If false positive:**
- Update health check configuration
- Adjust failure threshold
- No action needed on failover

---

### Scheduled Maintenance Window

**Frequency:** Monthly

**Checklist:**
- [ ] Test PITR restore (Member 2)
- [ ] Test on-demand backup (Member 2)
- [ ] Trigger test failover (Member 3)
- [ ] Measure RTO (Member 3)
- [ ] Update runbooks with lessons learned
- [ ] Review and update documentation

---

## Escalation Path

Level 1: On-call engineer (you)
- Responds to all alerts
- Resolves routine issues
- Escalates if needed

Level 2: Team Lead
- Complex issues
- Architecture decisions
- Approval for major changes

Level 3: Mentor/Manager
- Complete system failures
- Business-critical decisions
- Customer communication

## Contact Information
[Add team contact details]
