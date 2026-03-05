# Backup & Restore Runbook

## Daily Automated Backups
- Point-in-Time Recovery: Enabled on all tables
- Retention: 35 days
- No manual action required

## Weekly On-Demand Backups
Perform every Sunday at 23:00 UTC

### Steps:
1. Go to DynamoDB → Products-table → Backups
2. Create backup: `products-weekly-YYYY-MM-DD`
3. Repeat for VendorInventory and StockTransactions

## Restore Procedures

### Scenario 1: Accidental Table Deletion
**RTO: 10 minutes**

1. DynamoDB → Backups tab
2. Select latest backup
3. Click "Restore"
4. New table name: [original-name]-restored-TIMESTAMP
5. Restore indexes: Yes
6. Wait for "Active" status
7. Update Lambda environment variables to point to new table
8. Test APIs
9. If successful, delete old table and rename new table

### Scenario 2: Accidental Data Deletion
**RTO: 15 minutes**

1. Determine time of deletion
2. DynamoDB → Backups → Point-in-time recovery
3. Restore to: [time before deletion]
4. New table name: [original-name]-pitr-TIMESTAMP
5. Wait for "Active" status
6. Copy missing data from restored table to production table
7. Delete restored table

### Scenario 3: Data Corruption
**RTO: 15 minutes**

1. Identify corrupted data
2. Restore from Point-in-Time Recovery to before corruption
3. Export clean data
4. Import into production table
5. Verify data integrity

## S3 Backup
- Cross-Region Replication: Automatic
- Source: shopsmart-inventory-images (Mumbai)
- Destination: shopsmart-inventory-images-dr (Singapore)
- Replication lag: < 15 minutes

## Testing Schedule
- Monthly: Test PITR restore
- Quarterly: Test full table restore
- Annually: Test complete DR failover
