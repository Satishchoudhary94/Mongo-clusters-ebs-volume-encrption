
# 🔐 Production MongoDB Cluster EBS Encryption — Zero Downtime Guide

> **Cluster:** `rs0` | **Data Size:** 1.4TB per node | **Nodes:** 3 | **Region:** `ap-south-1`

---

## Production Cluster Reference

| Role | Instance ID | Private IP | DNS | Type | Volume ID | Size |
|---|---|---|---|---|---|---|
| **Primary** | `i-0809b75118a62876d` | `172.31.35.162` | `mongo.primary.prod.internal` | `r6g.8xlarge` | `vol-0efbeaa27c5653b61` | 1440 GB |
| **Secondary-0** | `i-0254ee6f9d8b6d747` | `172.31.28.104` | `mongo.secondary.0.prod.internal` | `t4g.2xlarge` | `vol-085330a35843d900a` | 1440 GB |
| **Secondary-1** | `i-09c17abc59286531d` | `172.31.12.162` | `mongo.secondary.1.prod.internal` | `t4g.2xlarge` | `vol-0525abc6fd16d83bb` | 1440 GB |

### Current RS Config

| Node | Priority | Votes | Can Become Primary? |
|---|---|---|---|
| `mongo.primary.prod.internal` | `1` | `1` | ✅ Yes |
| `mongo.secondary.0.prod.internal` | `0` | `1` | ❌ No |
| `mongo.secondary.1.prod.internal` | `0` | `0` | ❌ No |

> ⚠️ **Critical:** Both secondaries have `priority: 0` — they cannot become PRIMARY. We must update priority before stepping down the Primary. This is handled in Phase 3.

---

## Encryption Strategy — Oplog Based (No Full Sync!)

> ✅ Since volumes are 1.4TB, we use **fsyncLock + snapshot + local.\* delete** approach.
> This triggers **oplog-only sync** instead of full initial sync — much faster!

```
PHASE 1 → Encrypt Secondary-1 (priority:0, votes:0)  ← Safest, start here
           ↓ Verify RS healthy
PHASE 2 → Encrypt Secondary-0 (priority:0, votes:1)
           ↓ Verify RS healthy
PHASE 3 → Update priorities → Stepdown Primary → Encrypt Primary
           ↓ Verify RS fully healthy ✅
```

---

## PRE-REQUISITES — Run Before Starting

### 1. Verify RS is Fully Healthy

```bash
mongosh "mongodb://prod_Iuyf76:7sPvUitD8ro%40zVDCiRp4@mongo.primary.prod.internal:27017/reelo_prod_dkkBy?replicaSet=rs0&authSource=reelo_prod_dkkBy" \
  --eval "rs.status()" | grep -E 'name|stateStr|health'
```

All 3 nodes must show `health: 1` before proceeding.

### 2. Increase Oplog Size on ALL 3 Nodes (Safety Net)

Run on **each node** one by one:

```javascript
// Run inside mongosh on PRIMARY first, then each secondary
use local
db.adminCommand({ replSetResizeOplog: 1, size: Double(10240) })
```

Verify on all nodes:
```javascript
rs.printReplicationInfo()
```

Expected output per node:
```
configured oplog size: 10240 MB
```

---

## PHASE 1 — Encrypt Secondary-1 (`172.31.12.162`)

> Secondary-1 has `priority:0, votes:0` — safest node to start with.

### Step 1 — fsyncLock on Secondary-1

SSH into Secondary-1:
```bash
ssh -i production.pem ubuntu@172.31.12.162
```

Run inside mongosh:
```javascript
mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval "db.fsyncLock()"
```

Expected output:
```json
{ "info": "now locked against writes", "ok": 1 }
```

### Step 2 — Create Snapshot (From Another Terminal — Immediately!)

```bash
aws ec2 create-snapshot \
  --region ap-south-1 \
  --volume-id vol-0525abc6fd16d83bb \
  --description "pre-encryption-mongo-secondary1-prod-$(date +%Y%m%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=mongo-secondary1-prod-pre-encrypt}]'
```

> 📝 Note the `SnapshotId` → `snap-xxxxxxxxxxxxxxxxx`

### Step 3 — fsyncUnlock Immediately After Snapshot is Triggered

```javascript
db.fsyncUnlock()
```

Expected output:
```json
{ "info": "unlock completed", "ok": 1 }
```

> ⚠️ Never leave fsyncLock on for more than a few seconds!

### Step 4 — Wait for Snapshot to Complete

```bash
aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-xxxxxxxxxxxxxxxxx

echo "✅ Secondary-1 snapshot done!"
```

### Step 5 — Copy Snapshot With Encryption

```bash
aws ec2 copy-snapshot \
  --region ap-south-1 \
  --source-region ap-south-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "encrypted-mongo-secondary1-prod-$(date +%Y%m%d)"
```

> 📝 Note new encrypted `SnapshotId` → `snap-yyyyyyyyyyyyyyyyy`

Wait for encrypted snapshot:
```bash
aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-yyyyyyyyyyyyyyyyy

echo "✅ Encrypted snapshot done!"
```

### Step 6 — Register AMI

```bash
aws ec2 register-image \
  --region ap-south-1 \
  --name "mongo-secondary1-prod-encrypted-$(date +%Y%m%d)" \
  --architecture arm64 \
  --root-device-name /dev/sda1 \
  --virtualization-type hvm \
  --block-device-mappings '[
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "SnapshotId": "snap-yyyyyyyyyyyyyyyyy",
        "VolumeType": "gp3",
        "VolumeSize": 1440,
        "DeleteOnTermination": true
      }
    }
  ]'
```

> 📝 Note `ImageId` → `ami-zzzzzzzzzzzzzzzzz`

### Step 7 — Launch New Encrypted Secondary-1

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-zzzzzzzzzzzzzzzzz \
  --instance-type t4g.2xlarge \
  --key-name production \
  --security-group-ids sg-0c9b049a07440914f \
  --subnet-id subnet-dfa9f393 \
  --iam-instance-profile Arn=arn:aws:iam::840983897776:instance-profile/EC2SSMRole \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mongoDB-secondary-1-encrypted}]' \
  --count 1
```

Wait for running:
```bash
aws ec2 wait instance-running \
  --region ap-south-1 \
  --instance-ids i-NEW-SECONDARY1

echo "✅ New encrypted secondary-1 is running!"
```

### Step 8 — Delete local.* on New Instance (Triggers Oplog Sync!)

SSH into new instance:
```bash
ssh -i production.pem ubuntu@<new-secondary1-ip>

sudo systemctl stop mongod
sudo rm /var/lib/mongodb/local.*
sudo systemctl start mongod
sudo systemctl status mongod
```

> ✅ Deleting `local.*` forces MongoDB to do **oplog-only sync** instead of full initial sync!

### Step 9 — Add New Node to RS from PRIMARY

On PRIMARY (`172.31.35.162`):

```javascript
rs.add({
  host: '<new-secondary1-ip>:27017',
  priority: 0,
  votes: 0
})
```

Watch sync — should go straight to SECONDARY (no STARTUP2!):
```bash
watch -n 5 "mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval \"rs.status()\" | grep -E 'name|stateStr|health'"
```

Expected (fast sync via oplog):
```
mongo.primary.prod.internal       → PRIMARY    health: 1 ✅
mongo.secondary.0.prod.internal   → SECONDARY  health: 1 ✅
mongo.secondary.1.prod.internal   → SECONDARY  health: 1 ✅
<new-secondary1-ip>:27017         → SECONDARY  health: 1 ✅ (oplog syncing)
```

### Step 10 — Verify Replication Lag = 0

```javascript
rs.printSecondaryReplicationInfo()
```

Wait until new node shows:
```
0 secs (0 hrs) behind the primary
```

### Step 11 — Remove Old Secondary-1 & Terminate

On PRIMARY:
```javascript
rs.remove('mongo.secondary.1.prod.internal:27017')
```

```bash
aws ec2 terminate-instances \
  --region ap-south-1 \
  --instance-ids i-09c17abc59286531d

echo "✅ Old Secondary-1 terminated!"
```

### Step 12 — Verify RS Healthy (2+1 nodes)

```bash
mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval "rs.status()" | grep -E 'name|stateStr|health'
```

---

## PHASE 2 — Encrypt Secondary-0 (`172.31.28.104`)

> Same steps as Phase 1. Secondary-0 has `priority:0, votes:1`.

### Step 1 — fsyncLock on Secondary-0

```bash
ssh -i production.pem ubuntu@172.31.28.104

mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval "db.fsyncLock()"
```

### Step 2 — Create Snapshot

```bash
aws ec2 create-snapshot \
  --region ap-south-1 \
  --volume-id vol-085330a35843d900a \
  --description "pre-encryption-mongo-secondary0-prod-$(date +%Y%m%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=mongo-secondary0-prod-pre-encrypt}]'
```

### Step 3 — fsyncUnlock Immediately

```javascript
db.fsyncUnlock()
```

### Step 4 — Wait & Copy With Encryption

```bash
# Wait for original
aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-xxxxxxxxxxxxxxxxx

# Copy with encryption
aws ec2 copy-snapshot \
  --region ap-south-1 \
  --source-region ap-south-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "encrypted-mongo-secondary0-prod-$(date +%Y%m%d)"

# Wait for encrypted snapshot
aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-yyyyyyyyyyyyyyyyy
```

### Step 5 — Register AMI

```bash
aws ec2 register-image \
  --region ap-south-1 \
  --name "mongo-secondary0-prod-encrypted-$(date +%Y%m%d)" \
  --architecture arm64 \
  --root-device-name /dev/sda1 \
  --virtualization-type hvm \
  --block-device-mappings '[
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "SnapshotId": "snap-yyyyyyyyyyyyyyyyy",
        "VolumeType": "gp3",
        "VolumeSize": 1440,
        "DeleteOnTermination": true
      }
    }
  ]'
```

### Step 6 — Launch New Encrypted Secondary-0

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-zzzzzzzzzzzzzzzzz \
  --instance-type t4g.2xlarge \
  --key-name production \
  --security-group-ids sg-0783d659e3916ff73 \
  --subnet-id subnet-cc7f13b7 \
  --iam-instance-profile Arn=arn:aws:iam::840983897776:instance-profile/EC2SSMRole \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mongoDB-secondary-0-encrypted}]' \
  --count 1
```

### Step 7 — Delete local.* on New Instance

```bash
ssh -i production.pem ubuntu@<new-secondary0-ip>

sudo systemctl stop mongod
sudo rm /var/lib/mongodb/local.*
sudo systemctl start mongod
```

### Step 8 — Add to RS from PRIMARY

```javascript
rs.add({
  host: '<new-secondary0-ip>:27017',
  priority: 0,
  votes: 1
})
```

### Step 9 — Wait for Sync & Verify Lag = 0

```bash
watch -n 5 "mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval \"rs.status()\" | grep -E 'name|stateStr|health'"
```

```javascript
rs.printSecondaryReplicationInfo()
// Wait for: 0 secs behind the primary
```

### Step 10 — Remove Old Secondary-0 & Terminate

```javascript
rs.remove('mongo.secondary.0.prod.internal:27017')
```

```bash
aws ec2 terminate-instances \
  --region ap-south-1 \
  --instance-ids i-0254ee6f9d8b6d747
```

---

## PHASE 3 — Encrypt Primary (`172.31.35.162`)

> ⚠️ Most critical phase. Follow exact order below.

### Step 1 — Update RS Priorities (Critical for Smooth Stepdown!)

On PRIMARY (`172.31.35.162`), run inside mongosh:

```javascript
var cfg = rs.conf();

// Give new secondary-0 priority so it can become PRIMARY
cfg.members.forEach(function(m) {
  if (m.host.includes('<new-secondary0-ip>')) {
    m.priority = 1;  // Can now become PRIMARY
    m.votes = 1;
  }
  if (m.host.includes('<new-secondary1-ip>')) {
    m.priority = 1;  // Can now become PRIMARY
    m.votes = 1;
  }
  if (m.host === 'mongo.primary.prod.internal:27017') {
    m.priority = 0;  // Current primary steps back temporarily
  }
});

rs.reconfig(cfg);
```

Verify priorities:
```javascript
rs.conf().members.forEach(m => print(m.host, 'priority:', m.priority, 'votes:', m.votes))
```

### Step 2 — Verify Replication Lag = 0 Before Stepdown

```javascript
rs.printSecondaryReplicationInfo()
// Both secondaries must show: 0 secs behind the primary
```

### Step 3 — Smooth Stepdown (No Force Needed!)

```javascript
rs.stepDown(120)
```

Verify new PRIMARY elected:
```bash
mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval "rs.status()" | grep -E 'name|stateStr|health'
```

Expected:
```
172.31.35.162         → SECONDARY  health: 1 ✅ (old primary stepped down)
<new-secondary0-ip>   → PRIMARY    health: 1 ✅ (new primary elected)
<new-secondary1-ip>   → SECONDARY  health: 1 ✅
```

### Step 4 — fsyncLock on Old Primary (Now Secondary)

```bash
ssh -i production.pem ubuntu@172.31.35.162

mongosh -u prod_Iuyf76 -p '7sPvUitD8ro@zVDCiRp4' \
  --authenticationDatabase reelo_prod_dkkBy \
  --eval "db.fsyncLock()"
```

### Step 5 — Snapshot Old Primary Volume

```bash
aws ec2 create-snapshot \
  --region ap-south-1 \
  --volume-id vol-0efbeaa27c5653b61 \
  --description "pre-encryption-mongo-primary-prod-$(date +%Y%m%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=mongo-primary-prod-pre-encrypt}]'
```

### Step 6 — fsyncUnlock Immediately

```javascript
db.fsyncUnlock()
```

### Step 7 — Wait & Copy With Encryption

```bash
aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-xxxxxxxxxxxxxxxxx

aws ec2 copy-snapshot \
  --region ap-south-1 \
  --source-region ap-south-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "encrypted-mongo-primary-prod-$(date +%Y%m%d)"

aws ec2 wait snapshot-completed \
  --region ap-south-1 \
  --snapshot-ids snap-yyyyyyyyyyyyyyyyy
```

### Step 8 — Register AMI

```bash
aws ec2 register-image \
  --region ap-south-1 \
  --name "mongo-primary-prod-encrypted-$(date +%Y%m%d)" \
  --architecture arm64 \
  --root-device-name /dev/sda1 \
  --virtualization-type hvm \
  --block-device-mappings '[
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "SnapshotId": "snap-yyyyyyyyyyyyyyyyy",
        "VolumeType": "gp3",
        "VolumeSize": 1440,
        "DeleteOnTermination": true
      }
    }
  ]'
```

### Step 9 — Launch New Encrypted Primary Instance

```bash
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id ami-zzzzzzzzzzzzzzzzz \
  --instance-type r6g.8xlarge \
  --key-name production \
  --security-group-ids sg-0c9b049a07440914f \
  --subnet-id subnet-b747b2dc \
  --iam-instance-profile Arn=arn:aws:iam::840983897776:instance-profile/EC2SSMRole \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Mongodb-Primary-encrypted}]' \
  --count 1
```

### Step 10 — Delete local.* on New Primary Instance

```bash
ssh -i production.pem ubuntu@<new-primary-ip>

sudo systemctl stop mongod
sudo rm /var/lib/mongodb/local.*
sudo systemctl start mongod
```

### Step 11 — Add New Node to RS

On current PRIMARY (new secondary-0):

```javascript
rs.add({
  host: '<new-primary-ip>:27017',
  priority: 2,   // Highest priority — will become PRIMARY after stepdown
  votes: 1
})
```

### Step 12 — Wait for Full Sync

```bash
watch -n 5 "mongosh ... --eval \"rs.status()\" | grep -E 'name|stateStr|health'"
```

```javascript
rs.printSecondaryReplicationInfo()
// Wait for new node: 0 secs behind the primary
```

### Step 13 — Remove Old Primary from RS

```javascript
rs.remove('mongo.primary.prod.internal:27017')
```

```bash
aws ec2 terminate-instances \
  --region ap-south-1 \
  --instance-ids i-0809b75118a62876d
```

### Step 14 — Update Route 53 DNS Records

```bash
# Update all 3 DNS records to new IPs
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXXXXXXX \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "mongo.primary.prod.internal",
          "Type": "A",
          "TTL": 60,
          "ResourceRecords": [{"Value": "<new-primary-ip>"}]
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "mongo.secondary.0.prod.internal",
          "Type": "A",
          "TTL": 60,
          "ResourceRecords": [{"Value": "<new-secondary0-ip>"}]
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "mongo.secondary.1.prod.internal",
          "Type": "A",
          "TTL": 60,
          "ResourceRecords": [{"Value": "<new-secondary1-ip>"}]
        }
      }
    ]
  }'
```

---

## Final Verification

```bash
# Verify all volumes encrypted
aws ec2 describe-volumes \
  --region ap-south-1 \
  --filters "Name=attachment.instance-id,Values=<new-primary-id>,<new-secondary0-id>,<new-secondary1-id>" \
  --query 'Volumes[].{VolumeId:VolumeId,Encrypted:Encrypted,Size:Size,InstanceId:Attachments[0].InstanceId}'
```

Expected:
```json
[
  { "Encrypted": true, "Size": 1440 },
  { "Encrypted": true, "Size": 1440 },
  { "Encrypted": true, "Size": 1440 }
]
```

```bash
# Verify RS fully healthy
mongosh "mongodb://prod_Iuyf76:7sPvUitD8ro%40zVDCiRp4@mongo.primary.prod.internal:27017/reelo_prod_dkkBy?replicaSet=rs0&authSource=reelo_prod_dkkBy" \
  --eval "rs.status()" | grep -E 'name|stateStr|health'
```

---

## ID Tracker — Fill As You Go

### Phase 1 — Secondary-1

| Resource | Old | New |
|---|---|---|
| Instance ID | `i-09c17abc59286531d` | `i-` ← fill |
| Volume ID | `vol-0525abc6fd16d83bb` | — |
| Original Snapshot | — | `snap-` ← fill |
| Encrypted Snapshot | — | `snap-` ← fill |
| AMI ID | — | `ami-` ← fill |
| Private IP | `172.31.12.162` | ← fill |

### Phase 2 — Secondary-0

| Resource | Old | New |
|---|---|---|
| Instance ID | `i-0254ee6f9d8b6d747` | `i-` ← fill |
| Volume ID | `vol-085330a35843d900a` | — |
| Original Snapshot | — | `snap-` ← fill |
| Encrypted Snapshot | — | `snap-` ← fill |
| AMI ID | — | `ami-` ← fill |
| Private IP | `172.31.28.104` | ← fill |

### Phase 3 — Primary

| Resource | Old | New |
|---|---|---|
| Instance ID | `i-0809b75118a62876d` | `i-` ← fill |
| Volume ID | `vol-0efbeaa27c5653b61` | — |
| Original Snapshot | — | `snap-` ← fill |
| Encrypted Snapshot | — | `snap-` ← fill |
| AMI ID | — | `ami-` ← fill |
| Private IP | `172.31.35.162` | ← fill |

---

## Key Rules — Never Break These!

| Rule | Detail |
|---|---|
| ✅ One node at a time | Never encrypt 2 nodes simultaneously |
| ✅ fsyncUnlock immediately | Never leave fsyncLock for more than a few seconds |
| ✅ Verify RS healthy between phases | All nodes must show `health: 1` before next phase |
| ✅ Lag = 0 before stepdown | Always check `rs.printSecondaryReplicationInfo()` |
| ✅ Priority update before stepdown | Update priorities BEFORE calling `rs.stepDown()` |
| ✅ local.* delete on new nodes | Triggers oplog sync — avoids 1.4TB full resync! |
| ✅ Keep old instances 24-48hrs | Rollback possible if needed |
| ❌ Never force reconfig on new node | It will corrupt RS config |
| ❌ Never terminate before RS stable | Always verify RS healthy first |

---

## Rollback Plan

If anything goes wrong at any phase:

```bash
# Restore DNS to old IP
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXXXXXXX \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "mongo.primary.prod.internal",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<old-ip>"}]
      }
    }]
  }'
```

```javascript
// Add old node back to RS if needed
rs.add({ host: 'mongo.primary.prod.internal:27017', priority: 1, votes: 1 })
```

---

## Summary

```
PRE-REQ  → Verify RS healthy → Resize oplog to 10GB on all nodes

PHASE 1  → fsyncLock Secondary-1 → Snapshot → fsyncUnlock
           → Encrypt snapshot → Launch new node → Delete local.*
           → RS add (oplog sync) → Remove old Secondary-1

PHASE 2  → fsyncLock Secondary-0 → Snapshot → fsyncUnlock
           → Encrypt snapshot → Launch new node → Delete local.*
           → RS add (oplog sync) → Remove old Secondary-0

PHASE 3  → Update priorities → rs.stepDown() (smooth!)
           → fsyncLock old Primary → Snapshot → fsyncUnlock
           → Encrypt snapshot → Launch new node → Delete local.*
           → RS add (oplog sync) → Remove old Primary
           → Update Route 53 DNS for all 3 nodes

VERIFY   → All 3 volumes Encrypted: true ✅
           → RS healthy: all nodes health: 1 ✅
           → Zero downtime throughout ✅
```
