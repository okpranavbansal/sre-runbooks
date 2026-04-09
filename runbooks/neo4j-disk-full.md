# Runbook: Neo4j Disk Full

**Severity:** P1 — Neo4j becomes read-only at 95% disk, write queries fail  
**Alert:** `CloudWatch EBS VolumeSpaceAvailable < 10GB on neo4j data volume`

---

## Infrastructure Context

Neo4j runs on an EC2 ASG (`min=0, max=1, desired=1`) with:
- OS disk: 30GB gp3 (`/dev/sda1`)
- Data disk: 100GB gp3 (`/dev/sdf`, mounted at `/data`)

Both volumes are encrypted. The data volume has `delete_on_termination=false` to survive instance replacement.

---

## Symptoms

- Neo4j write queries returning `DatabaseException: store can not be written to`
- Application services returning 500 on graph DB operations
- CloudWatch alarm `Neo4jDiskCritical` firing

---

## Immediate Steps (< 10 minutes)

```bash
# 1. SSH to Neo4j instance (via SSM — no bastion needed)
aws ssm start-session --target <instance-id>

# 2. Check disk usage
df -h /data
du -sh /data/neo4j/data/databases/*/

# 3. Check for large transaction logs (these accumulate and are safe to archive)
ls -lh /data/neo4j/data/transactions/

# 4. Check Neo4j status
systemctl status neo4j
```

---

## Quick Relief: Clear Transaction Logs

Transaction logs older than the retention period are safe to delete.

```bash
# Neo4j must be stopped to safely move logs
sudo systemctl stop neo4j

# Archive old transaction logs (keep last 24h)
find /data/neo4j/data/transactions -name "*.log" -mtime +1 \
  -exec mv {} /tmp/neo4j-txlogs-archive/ \;

# Or delete if not needed for point-in-time recovery
find /data/neo4j/data/transactions -name "*.log" -mtime +7 -delete

sudo systemctl start neo4j
```

---

## Permanent Fix: Expand EBS Volume

EBS gp3 volumes can be expanded online (no downtime for the volume attachment, but the filesystem must be resized).

```bash
# 1. Get the volume ID
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[?DeviceName==`/dev/sdf`].Ebs.VolumeId' \
  --output text

# 2. Expand the volume (e.g. 100GB → 200GB)
aws ec2 modify-volume \
  --volume-id vol-PLACEHOLDER \
  --size 200

# 3. Wait for volume modification to complete
aws ec2 describe-volumes-modifications --volume-ids vol-PLACEHOLDER

# 4. Expand the filesystem online (for ext4)
sudo growpart /dev/nvme1n1 1   # adjust device name as needed
sudo resize2fs /dev/nvme1n1p1

# 5. Verify
df -h /data
```

---

## Terraform Update (to make permanent)

Update the Terraform variable and re-apply to make the new size the source of truth:

```hcl
# applications/neo4j/main.tf — update volume_size
block_device_mappings = [
  {
    device_name = "/dev/sdf"
    ebs = {
      delete_on_termination = false
      encrypted             = true
      volume_size           = 200   # was 100
      volume_type           = "gp3"
    }
  }
]
```

---

## Prevention

- [ ] Alert at 70% disk usage (not 95%)
- [ ] Configure Neo4j `dbms.tx_log.rotation.retention_policy=7 days` (prevents unbounded log growth)
- [ ] Weekly automated backup to S3 with log pruning post-backup
- [ ] Set volume size in Terraform to 200GB from the start; EBS is cheap
