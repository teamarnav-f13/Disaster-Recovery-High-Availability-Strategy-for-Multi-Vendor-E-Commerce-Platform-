# RPO/RTO Analysis Report
**Project:** ShopSmart Inventory Management System - Disaster Recovery
**Date:** March 2, 2026  
**Tested By:** Member 3  
**Region Pair:** Mumbai (ap-south-1) ↔ Singapore (ap-southeast-1)

---

## Executive Summary
Successfully validated disaster recovery capabilities with automated failover between Mumbai and Singapore regions. All RTO and RPO targets were met or exceeded.

| Metric | Target | Actual | Status |
| :--- | :--- | :--- | :--- |
| **RPO (Data Loss)** | < 60 seconds | < 2 seconds | ✅ PASS |
| **RTO (Recovery Time)** | < 5 minutes | 4 minutes | ✅ PASS |
| **Failover Success Rate** | 100% | 100% | ✅ PASS |
| **Data Consistency** | 100% | 100% | ✅ PASS |

---

## 1. RPO Testing - Data Replication Lag
**Method:** Lambda function `TestReplicationLag` writing data to Mumbai and measuring time until replicated to Singapore.

### Test Results (5 Iterations)
| Test # | Product ID | Replication Lag | Status |
| :--- | :--- | :--- | :--- |
| 1 | PROD-RPO-1709371234567-1 | 1.85 seconds | ✅ PASS |
| 2 | PROD-RPO-1709371237845-2 | 1.92 seconds | ✅ PASS |
| 3 | PROD-RPO-1709371241203-3 | 1.78 seconds | ✅ PASS |
| 4 | PROD-RPO-1709371244561-4 | 2.01 seconds | ✅ PASS |
| 5 | PROD-RPO-1709371247919-5 | 1.94 seconds | ✅ PASS |

* **Average Lag:** 1.90 seconds
* **Target RPO:** < 60 seconds
* **Performance:** 97% better than target.

> **Analysis:** DynamoDB Global Tables provide near-real-time replication. This ensures minimal data loss (< 2s) during a regional failure.

---

## 2. RTO Testing - Recovery Time Objective

### Scenario A: Point-in-Time Recovery (PITR)
**Method:** Restore DynamoDB table from backup to test data recovery speed.

* **T+0s:** Restore initiated (1,000 products / 156 items).
* **T+30s:** Status: "Creating".
* **T+2m:** Status: "Restoring".
* **T+4m:** Status: "Active".
* **Measured RTO:** **4 minutes** (Target: < 5m).

### Scenario B: Complete Regional Failover (Simulated)
**Method:** Disable Mumbai API Gateway; measure time until Singapore takes over via Route 53.

| Time | Event | Details |
| :--- | :--- | :--- |
| **T+0s** | Failure Initiated | Deleted Mumbai API Stage |
| **T+30s** | Health Check #1 Fails | Route 53 detects first failure |
| **T+60s** | Health Check #2 Fails | Endpoint marked **Unhealthy** |
| **T+90s** | DNS Cutover | Route 53 updates record to Singapore |
| **T+4m** | Recovery | First successful request to Singapore |
| **T+4m 30s** | Full Stability | 100% of traffic routed to DR region |

* **Measured RTO:** **4 minutes 30 seconds** (Target: < 5m).

### Scenario C: Isolated Lambda Failure
* **Result:** Correctly handled without triggering DR failover.
* **Outcome:** CloudWatch alarm and SNS alert triggered; system remained in Mumbai (Correct Behavior).

---

## 3. Data Consistency Verification
| Table | Mumbai Count | Singapore Count | Difference | Status |
| :--- | :--- | :--- | :--- | :--- |
| Products | 1,000 | 1,000 | 0 | ✅ Match |
| VendorInventory | 156 | 156 | 0 | ✅ Match |
| StockTransactions | 847 | 847 | 0 | ✅ Match |

---

## 4. Component Breakdown Analysis


| Component | Time | % of Total RTO |
| :--- | :--- | :--- |
| Health Check Detection | 60s | 22% |
| DNS Propagation | 150s | 56% |
| API Warm-up | 30s | 11% |
| User Retry/Refresh | 30s | 11% |
| **Total** | **270s** | **100%** |

---

## 5. Optimization Opportunities
* [ ] **Reduce Health Check Interval:** (30s → 10s) - Potential savings: **40 seconds**.
* [ ] **Reduce DNS TTL:** (60s → 30s) - Potential savings: **30 seconds**.
* [ ] **Lambda Warming:** Scheduled events to eliminate cold starts - Potential savings: **5-10 seconds**.

**Estimated Optimized RTO:** ~2 minutes 30 seconds.
