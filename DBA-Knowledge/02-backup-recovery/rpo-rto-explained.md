# RPO & RTO - Backup Recovery Objectives

Understanding RPO and RTO is fundamental to designing any backup and disaster recovery strategy.

## 🎯 RPO (Recovery Point Objective)

**"Maximum acceptable data loss"**

Measured in TIME: minutes/hours/days

### Example: RPO = 1 hour

```
Can lose up to 1 hour of transactions
Need backups at least hourly
Or continuous replication
```

### What This Means

- If a disaster strikes at 3:45 PM, and your last backup was at 3:00 PM
- You lose 45 minutes of data (acceptable if RPO = 1 hour)
- You lose the entire 45 minutes (unacceptable if RPO = 10 minutes)

### How to Achieve RPO

| RPO Target    | Approach                              | Cost      |
| ------------- | ------------------------------------- | --------- |
| 24 hours      | Daily backups                         | Low       |
| 1-4 hours     | Multiple daily backups                | Medium    |
| 15-30 minutes | Hourly backups + transaction logs     | Medium    |
| < 15 minutes  | Continuous replication (WAL shipping) | High      |
| Near-zero     | Synchronous replication               | Very High |

## ⏱️ RTO (Recovery Time Objective)

**"Maximum acceptable downtime"**

Measured in TIME to restore service

### Example: RTO = 30 minutes

```
Database must be online within 30 min after failure
Need automated failover OR fast restore procedure
Requires backup accessible and tested
```

### What This Means

- Disaster happens at 3:00 PM
- Business impact starts immediately (revenue loss, customer impact, etc.)
- By 3:30 PM, database MUST be online
- Everything from notification to restore must take ≤ 30 minutes

### How to Achieve RTO

| RTO Target    | Approach                           | Cost      | Setup          |
| ------------- | ---------------------------------- | --------- | -------------- |
| 4-8 hours     | Manual restore from backup         | Low       | Easy           |
| 1-2 hours     | Automated restore + warm standby   | Medium    | Moderate       |
| 15-30 minutes | HA failover to replica             | High      | Complex        |
| < 5 minutes   | Active-active + load balancer      | Very High | Very Complex   |
| < 1 minute    | Synchronous replication + auto-DNS | Extreme   | Highly Complex |

## 🔀 RPO vs RTO - The Trade-off

```
RPO ← DATA LOSS     vs     UPTIME → RTO
│
↑ Tight RPO = More expensive
  (continuous backup/replication)

                    ↑ Tight RTO = More expensive
                      (HA/failover automation)

Business decides acceptable trade-off
```

## 📊 Business Impact Examples

### Example 1: E-commerce Platform

```
Requirement: 99.99% uptime (52 minutes/year downtime allowed)

Business Impact of 1 hour downtime:
- $50,000/hour revenue loss
- Reputation damage
- Customer complaints

Design:
- RPO: 5 minutes (hourly backups + continuous replication)
- RTO: 10 minutes (automated failover to hot standby)
- Cost: $500k/year for infrastructure
```

### Example 2: Internal Reporting Database

```
Requirement: 95% uptime (21.6 hours/year downtime allowed)

Business Impact of 4 hours downtime:
- Reports delayed, no revenue impact
- Teams can work offline
- Acceptable

Design:
- RPO: 24 hours (daily backups only)
- RTO: 4 hours (manual restore, business hours)
- Cost: $20k/year for infrastructure
```

### Example 3: Regulatory Compliance Database

```
Requirement: Meet PCI-DSS, must recover within 1 hour

Business Impact of 1 hour downtime:
- Transactions queued, can retry
- Compliance requirement: 99.99% uptime

Design:
- RPO: 15 minutes (transaction log backups)
- RTO: 45 minutes (automated restore with testing)
- Cost: $300k/year for infrastructure
```

## 🎯 How to Calculate Costs

### RPO Cost Calculation

```
Tighter RPO = More frequent backups = More:
- Backup storage
- Network bandwidth
- Backup software licenses
- Replication overhead
```

**Formula:** Total_Cost = Backup_Frequency × Storage_Cost × Retention_Period

### RTO Cost Calculation

```
Tighter RTO = More sophisticated infrastructure = More:
- Redundant systems
- Replication setup
- Automated failover tools
- Monitoring & alerting
```

**Formula:** Total_Cost = Redundancy_Level × Infrastructure_Cost

## ✅ Checklist: Align RPO/RTO with Business

- [ ] Meet with business stakeholders
- [ ] Document maximum acceptable data loss (RPO)
- [ ] Document maximum acceptable downtime (RTO)
- [ ] Calculate business impact per hour of downtime
- [ ] Calculate cost of achieving RPO/RTO
- [ ] Verify cost is acceptable (usually < business impact cost)
- [ ] Document in disaster recovery plan
- [ ] Test recovery procedures quarterly
- [ ] Update as business requirements change

## 🔗 Related Topics

- [Backup Strategies](backup-strategies.md) — How to achieve your RPO
- [Restore Testing](restore-testing.md) — Verify your RTO is realistic
- [HA & Replication](../03-ha-replication/) — Alternative to backup for achieving tight RTO

---

**Key Insight:** RPO and RTO are business decisions, not technical decisions. Technology implements the decision, but the decision comes from understanding acceptable trade-offs.
