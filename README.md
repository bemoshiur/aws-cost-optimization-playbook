# AWS Cost Optimization & Migration Playbook

A practical, **read-only** checklist for finding wasted AWS spend and planning a safe migration to AWS. Every command below only *describes* or *gets* data; nothing here changes your account.

Maintained by **[S M Moshiur Rahman](https://github.com/bemoshiur)**, AWS Certified Solutions Architect (Dhaka, Bangladesh).
Want it done for you? **WhatsApp [+880 1777-174791](https://wa.me/8801777174791?text=Hi%20Moshiur%2C%20I%20used%20your%20AWS%20playbook%20and%20need%20help.)** · **[LinkedIn](https://www.linkedin.com/in/in-moshiur/)**

> Prerequisites: AWS CLI v2 configured with a role that has read-only access (the managed `ReadOnlyAccess` and `job-function/Billing` policies are enough). Run region-specific commands once per region you use. Use `--profile` if you have several accounts.

---

## Part 1: Cost optimization checklist

### Step 1. See where the money goes

```bash
# Last full month, grouped by service (replace the dates)
aws ce get-cost-and-usage \
  --time-period Start=2026-10-01,End=2026-11-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]' \
  --output text | sort -k2 -nr | head -15
```

Fix the top 3 line items first. They usually account for most of the bill.

### Step 2. Find idle and orphaned resources

```bash
# Unattached EBS volumes (you pay for them while "available")
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].[VolumeId,Size,VolumeType,CreateTime]' --output table

# Elastic IPs not attached to anything (public IPv4 addresses are billed)
aws ec2 describe-addresses \
  --query 'Addresses[?!AssociationId].[PublicIp,AllocationId]' --output table

# Your own snapshots, oldest first
aws ec2 describe-snapshots --owner-ids self \
  --query 'sort_by(Snapshots,&StartTime)[].[SnapshotId,StartTime,VolumeSize]' --output table

# NAT Gateways (hourly charge plus per-GB processing)
aws ec2 describe-nat-gateways --filter Name=state,Values=available \
  --query 'NatGateways[].[NatGatewayId,VpcId,SubnetId]' --output table

# Load balancers: check each one for traffic in CloudWatch before keeping it
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[].[LoadBalancerName,Type,CreatedTime]' --output table

# CloudWatch log groups that never expire
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].[logGroupName,storedBytes]' --output table
```

### Step 3. Right-size compute

```bash
# Cost Explorer rightsizing recommendations for EC2
aws ce get-rightsizing-recommendation --service AmazonEC2

# Compute Optimizer (opt in first: aws compute-optimizer update-enrollment-status --status Active)
aws compute-optimizer get-ec2-instance-recommendations
```

Also consider: Graviton (ARM) instances for supported workloads, gp3 instead of gp2 volumes, and Lambda memory tuning.

### Step 4. Commit smartly (Savings Plans / Reserved Instances)

Only commit to steady-state usage you have already measured.

```bash
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS
```

### Step 5. Storage and data transfer

- Put an S3 lifecycle policy on large buckets (transition to Infrequent Access / Glacier, expire old versions and incomplete multipart uploads), or use S3 Intelligent-Tiering for unpredictable access.
- Add **VPC gateway endpoints** for S3 and DynamoDB. They are free and keep that traffic off the NAT Gateway.
- Use CloudFront in front of public content to reduce origin data transfer.

### Step 6. Make savings stick

- **Tag everything** (`owner`, `env`, `project`) and activate the tags as cost allocation tags.
- Create **AWS Budgets** with alerts at 50%, 80% and 100% of the monthly budget.
- Turn on **Cost Anomaly Detection** (`aws ce create-anomaly-monitor`) so spikes reach you within a day.
- Review the bill monthly. Keep a short log of what changed and what it saved.

---

## Part 2: Migration to AWS runbook

### Phase 1. Assess (week 1)

- Inventory servers, databases, storage, dependencies, traffic, and compliance needs.
- Classify each workload: **rehost** (lift and shift), **replatform** (e.g. self-managed DB to RDS), **refactor** (e.g. to serverless/containers), **retire**, or **retain**.
- Record current monthly cost and performance baselines so you can prove the result.

### Phase 2. Design

- Target architecture: accounts and VPC layout, IAM, encryption (KMS), backups, monitoring, DR targets (RPO/RTO).
- Infrastructure as code from day one (AWS CDK, SST or Terraform) so environments are repeatable.
- Estimate run cost with the AWS Pricing Calculator before you build.

### Phase 3. Build and rehearse

- Stand up a landing zone and a staging copy of the workload.
- Migrate data with the right tool: AWS DMS (databases), AWS Application Migration Service (servers), `aws s3 sync` / DataSync (files).
- Load test, run security checks, and rehearse the cutover end to end.

### Phase 4. Cut over

- Lower DNS TTLs in advance. Freeze writes, run the final sync, switch DNS, verify.
- Keep the old environment running (read-only) with a **tested rollback** until the new one is proven.

### Phase 5. Optimize

- After 2–4 weeks of real traffic, right-size, apply Savings Plans, and tighten alarms and budgets (see Part 1).

---

## Need a hand?

If you'd rather have an experienced AWS architect review your account or plan your migration, message me:

- **WhatsApp:** [+880 1777-174791](https://wa.me/8801777174791?text=Hi%20Moshiur%2C%20I%20used%20your%20AWS%20playbook%20and%20need%20help.)
- **LinkedIn:** [linkedin.com/in/in-moshiur](https://www.linkedin.com/in/in-moshiur/)
- **Company:** [TICON SYSTEM LIMITED](https://www.ticonsys.com/)

Contributions and corrections are welcome via issues and pull requests.

<sub>Keywords: AWS cost optimization checklist · reduce AWS bill · AWS FinOps · AWS migration plan · lift and shift to AWS · AWS CLI cost audit · AWS Savings Plans · AWS consultant Bangladesh</sub>
