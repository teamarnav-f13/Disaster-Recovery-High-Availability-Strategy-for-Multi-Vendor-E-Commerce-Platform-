# Cost Analysis: Multi-Region Disaster Recovery (DR)

This document provides a breakdown of the monthly infrastructure costs for the Disaster Recovery setup between the **Mumbai (ap-south-1)** and **Singapore (ap-southeast-1)** regions.

---

## Monthly Cost Breakdown

| Service | Mumbai (Primary) | Singapore (DR) | Total Monthly Cost |
| :--- | :--- | :--- | :--- |
| **API Gateway** | $3.50 | $3.50 | $7.00 |
| **Lambda (idle)** | $0.00 | $0.00 | $0.00 |
| **DynamoDB Global Tables** | Included in Base | Included in Base | $25.00 |
| **S3 Cross-Region Replication** | $2.00 | $2.00 | $4.00 |
| **Route 53 Health Checks** | — | — | $1.00 |
| **CloudWatch Alarms (21)** | — | — | $2.10 |
| **SNS Notifications** | — | — | $0.50 |
| **Grand Total** | | | **$39.60** |

---

## Cost Observations

* **Lambda Efficiency:** Since Lambda is a compute-on-demand service, the idle cost remains **$0.00** in both regions. You only pay when traffic is actually routed to them.
* **Data Heaviness:** **DynamoDB Global Tables** represents the largest portion of the cost ($25.00). This is due to the overhead of maintaining multi-region replication and storage synchronization.
* **Monitoring Costs:** CloudWatch and Route 53 costs are fixed monthly fees based on the number of health checks and alarms configured (21 alarms total).
* **Storage Traffic:** S3 costs include the replication data transfer fees between the Mumbai and Singapore buckets.

---
*Last Updated: March 2026*
