# DR Runbook — High Availability & Disaster Recovery

> This runbook defines the operational procedures for failover, failback, and data recovery in this architecture.  
> It should be reviewed and tested quarterly. Last review: [date].

---

## Severity Classification

| Level | Description | Example |
|---|---|---|
| **P1 — Region Failure** | Primary AWS region fully unavailable | AWS us-east-1 outage |
| **P2 — AZ Failure** | One Availability Zone degraded or unavailable | Single AZ network disruption |
| **P3 — Service Failure** | Individual component failure within a healthy region | RDS instance crash, EC2 failure |
| **P4 — Data Recovery** | No active failure; retrieval of archived data needed | Glacier restore request |

---

## P2 — Availability Zone Failure

**Expected behavior:** Multi-AZ automatically handles this. No manual intervention required in most cases.

### Verify automatic failover completed

```bash
# Check RDS instance status
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,MultiAZ,AvailabilityZone]' \
  --output table

# Check current primary AZ
aws rds describe-db-instances \
  --db-instance-identifier <your-db-id> \
  --query 'DBInstances[0].AvailabilityZone'
```

### Verify Load Balancer health

```bash
# Check target group health — unhealthy nodes should have been removed
aws elbv2 describe-target-health \
  --target-group-arn <your-target-group-arn>
```

### Expected resolution time
- RDS Multi-AZ failover: typically under 2 minutes
- ELB removes unhealthy targets: typically under 30 seconds (based on health check interval)

---

## P1 — Region Failure (Full DR Failover)

> **This procedure results in promoting the Cross-Region Replica to primary.**  
> Execute only after confirming the primary region is genuinely unavailable, not just degraded.

### Step 1 — Confirm region failure

```bash
# Check AWS Service Health Dashboard
# https://health.aws.amazon.com/health/status

# Attempt to reach primary region endpoint
aws rds describe-db-instances --region <primary-region>
# If this times out or returns errors consistently for 5+ minutes, proceed
```

### Step 2 — Promote Cross-Region Replica to Primary

```bash
# Promote the read replica — this breaks replication and makes it writable
aws rds promote-read-replica \
  --db-instance-identifier <replica-db-id> \
  --region <dr-region>

# Wait for promotion to complete (status: available)
aws rds wait db-instance-available \
  --db-instance-identifier <replica-db-id> \
  --region <dr-region>
```

> ⚠️ **Note on data loss:** The replica may be behind the primary by seconds to minutes depending on replication lag at the time of failure. Any writes after the last replicated transaction will not be recoverable from the replica.

### Step 3 — Update DNS via Route 53

```bash
# Update the CNAME/A record pointing to the database endpoint
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.yourdomain.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<dr-region-db-endpoint>"}]
      }
    }]
  }'
```

### Step 4 — Redirect application traffic to DR region

```bash
# Update Route 53 to point application load balancer to DR region
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "<dr-elb-hosted-zone-id>",
          "DNSName": "<dr-elb-dns-name>",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 5 — Validate DR environment

```bash
# Verify database connectivity from DR worker nodes
# Run a lightweight read query against the promoted replica
psql -h <dr-region-db-endpoint> -U <user> -d <database> -c "SELECT 1;"

# Verify Queue is operational in DR region
aws sqs get-queue-attributes \
  --queue-url <dr-queue-url> \
  --attribute-names All \
  --region <dr-region>

# Verify S3 bucket accessibility
aws s3 ls s3://<bucket-name> --region <dr-region>
```

### Step 6 — Confirm service restoration

- [ ] API Gateway returning 200s
- [ ] Load Balancer showing all targets healthy
- [ ] Queue processing resumed (monitor CloudWatch: `NumberOfMessagesSent`, `NumberOfMessagesDeleted`)
- [ ] Database connections stable
- [ ] Analytics pipeline resuming normally

### Expected RTO
- Manual failover execution: 15–30 minutes (human decision + AWS propagation)
- DNS TTL propagation: up to 60 seconds (assuming TTL = 60)

---

## Failback — Returning to Primary Region

> Execute only after the primary region is confirmed stable and healthy.

### Step 1 — Restore primary database

```bash
# Create a new read replica in the primary region from the current DR primary
aws rds create-db-instance-read-replica \
  --db-instance-identifier <new-primary-replica-id> \
  --source-db-instance-identifier <dr-db-id> \
  --region <primary-region>

# Wait for replica to sync and reach "available" status
aws rds wait db-instance-available \
  --db-instance-identifier <new-primary-replica-id> \
  --region <primary-region>
```

### Step 2 — Validate data consistency

```bash
# Compare row counts on critical tables between DR primary and new primary replica
# Run checksums or record counts on key tables before promoting
psql -h <primary-replica-endpoint> -c "SELECT COUNT(*) FROM <critical_table>;"
psql -h <dr-db-endpoint> -c "SELECT COUNT(*) FROM <critical_table>;"
```

### Step 3 — Promote replica and redirect traffic

Repeat Steps 2–5 from the P1 Failover procedure, targeting the primary region.

### Step 4 — Re-establish Cross-Region Replica in DR region

```bash
# Re-create the DR replica from the restored primary
aws rds create-db-instance-read-replica \
  --db-instance-identifier <dr-replica-id> \
  --source-db-instance-identifier <primary-db-id> \
  --region <dr-region>
```

---

## P4 — Glacier Data Recovery

### Step 1 — Identify objects to restore

```bash
# List objects in a specific S3 prefix
aws s3 ls s3://<bucket-name>/<prefix>/ --recursive

# Check storage class (GLACIER objects need restoration)
aws s3api list-objects-v2 \
  --bucket <bucket-name> \
  --prefix <prefix> \
  --query 'Contents[?StorageClass==`GLACIER`].[Key,Size]' \
  --output table
```

### Step 2 — Initiate restore request

```bash
# Standard retrieval (3-5 hours) — recommended for non-urgent recovery
aws s3api restore-object \
  --bucket <bucket-name> \
  --key <object-key> \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}'

# Expedited retrieval (1-5 minutes) — use only when urgency justifies higher cost
aws s3api restore-object \
  --bucket <bucket-name> \
  --key <object-key> \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Expedited"}}'

# Bulk retrieval (5-12 hours) — use for large datasets where time is not critical
aws s3api restore-object \
  --bucket <bucket-name> \
  --key <object-key> \
  --restore-request '{"Days":1,"GlacierJobParameters":{"Tier":"Bulk"}}'
```

### Step 3 — Monitor restore status

```bash
# Check if restore is complete (look for "ongoing-request":"false")
aws s3api head-object \
  --bucket <bucket-name> \
  --key <object-key> \
  --query 'Restore'
```

### Step 4 — Access restored data

Once restored, the object is temporarily available in S3 for the number of days specified (`Days` parameter). Access it normally via S3 APIs or download via CLI:

```bash
aws s3 cp s3://<bucket-name>/<object-key> ./<local-filename>
```

### Step 5 — Monitor retrieval costs

```bash
# Review Glacier retrieval charges in Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=<YYYY-MM-DD>,End=<YYYY-MM-DD> \
  --granularity DAILY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Glacier"]}}' \
  --metrics BlendedCost
```

### Retrieval cost reference

| Tier | Time | Cost (approximate) |
|---|---|---|
| Expedited | 1–5 min | ~$0.03/GB + $0.01/request |
| Standard | 3–5 hours | ~$0.01/GB + $0.05/1000 requests |
| Bulk | 5–12 hours | ~$0.0025/GB + $0.025/1000 requests |

> Costs vary by region. Always verify current pricing at [aws.amazon.com/glacier/pricing](https://aws.amazon.com/glacier/pricing).

---

## Quarterly DR Test Checklist

Run this checklist every quarter to validate that the DR procedure works before it's needed in production.

- [ ] Simulate AZ failure and confirm Multi-AZ auto-failover completes within SLA
- [ ] Execute Cross-Region Replica promotion in a non-production environment
- [ ] Validate DNS failover propagation time
- [ ] Test Glacier restore for at least one object (Standard tier)
- [ ] Verify CloudWatch alarms fire correctly for RDS failover events
- [ ] Review and update this runbook if any procedures have changed
- [ ] Update estimated RTO/RPO based on actual test results

---

## Key Contacts & Resources

| Resource | Link |
|---|---|
| AWS Service Health Dashboard | https://health.aws.amazon.com/health/status |
| RDS Multi-AZ documentation | https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html |
| S3 Lifecycle documentation | https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html |
| Glacier pricing | https://aws.amazon.com/glacier/pricing |
| Route 53 failover routing | https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html |

---

*Maintained by Paulo Ricardo Potter*
