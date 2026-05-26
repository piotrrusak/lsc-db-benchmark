# YugabyteDB AWS Benchmark Runbook

## Architecture

Three EC2 instances (`10.0.0.11`, `10.0.0.12`, `10.0.0.13`), each running one YugabyteDB node, deployed in the same VPC. The benchmark client runs from a fourth instance (`10.0.0.14`) in the same VPC.

## Step 1 — Launch EC2 Instances

Launch **4 EC2 instances** (3 database nodes + 1 client) with the following configuration:

| Setting | Value |
|---|---|
| AMI | Ubuntu 22.04 |
| Instance type | `t3.medium` |
| VPC | Same VPC for all 4 instances |
| Key pair | Same key pair for all instances |
| Storage | At least 20 GB gp3 |

Private IPs used throughout this runbook:

| Role | IP |
|---|---|
| Node 1 | `10.0.0.11` |
| Node 2 | `10.0.0.12` |
| Node 3 | `10.0.0.13` |
| Client | `10.0.0.14` |

## Step 2 — Configure Security Group

Create or modify a security group and add the following **inbound rules**:

| Type | Protocol | Port range | Source | Purpose |
|---|---|---|---|---|
| Custom TCP | TCP | 1500 - 15000 | 0.0.0.0/0 | Basically everything |
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |

Apply this security group to all 4 instances.

---

## Phase 1 — Client Machine Environment Setup (Execute on Client: 10.0.0.14)

Update repositories and install the PostgreSQL 16 tool bundle (contains pgbench):

```bash
sudo apt update && sudo apt install -y postgresql-16
```
The transaction file

```sql
\set aid random(1, 100000 * :scale)
\set delta random(-5000, 5000)
BEGIN;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
END;
```

---

## Phase 3 — Cluster Startup Sequence

**On node 1** (`10.0.0.11`):

```bash
./bin/yugabyted start \
  --advertise_address=10.0.0.11 \
  --cloud_location=aws.us-east-1.us-east-1a
```

**On node 2** (`10.0.0.12`):

```bash
./bin/yugabyted start \
  --advertise_address=10.0.0.12 \
  --join=10.0.0.11 \
  --cloud_location=aws.us-east-1.us-east-1b
```

**On node 3** (`10.0.0.13`):

```bash
./bin/yugabyted start \
  --advertise_address=10.0.0.13 \
  --join=10.0.0.11 \
  --cloud_location=aws.us-east-1.us-east-1c
```

Verify all nodes are registered (from any node):

```bash
./bin/ysqlsh -h 10.0.0.11 -c "SELECT host, port, node_type FROM yb_servers();"
```

Expected output:

```
    host     | port | node_type
-------------+------+-----------
 10.0.0.11   | 5433 | primary
 10.0.0.12   | 5433 | primary
 10.0.0.13   | 5433 | primary
(3 rows)
```

---

## Phase 4 — Baseline Benchmark (Execute on Client: 10.0.0.14)

Provision the target database:

```bash
./bin/ysqlsh -h 10.0.0.11 -c "CREATE DATABASE benchmark;"
```

Generate 1,000,000 baseline rows using client-side data generation:

```bash
pgbench -i -s 10 -h 10.0.0.11 -p 5433 -U yugabyte -d benchmark --init-steps=dtgpf
```

Execute the baseline test and pipe logs to a file:

```bash
PGOPTIONS="-c client_min_messages=warning" pgbench -c 8 -j 4 -T 120 -P 1 -f ~/serializable_txn.sql -h 10.0.0.11 -p 5433 -U yugabyte benchmark > yb_results.log 2>&1
```

---

## Phase 6 — Network Partition Chaos Test

**[EXECUTE ON CLIENT — 10.0.0.14] Start Workload Engine**

Launch the transaction client in the background while streaming output to terminal and log file simultaneously:

```bash
PGOPTIONS="-c client_min_messages=warning" pgbench -c 8 -j 4 -T 120 -P 1 -f ~/serializable_txn.sql -h 10.0.0.11 -p 5433 -U yugabyte benchmark 2>&1 | tee yb_partition_results.log &
```

**[EXECUTE ON NODE 3 — 10.0.0.13 AT SECOND 30] Inject Partition Failure**

Run these lines exactly 30 seconds after the pgbench workload starts:

```bash
sudo iptables -A INPUT -s 10.0.0.11 -j DROP
sudo iptables -A INPUT -s 10.0.0.12 -j DROP
sudo iptables -A OUTPUT -d 10.0.0.11 -j DROP
sudo iptables -A OUTPUT -d 10.0.0.12 -j DROP
```

**[EXECUTE ON NODE 3 — 10.0.0.13 AT SECOND 60] Heal Network Partition**

Run this line exactly 60 seconds after the pgbench workload starts to re-join the cluster:

```bash
sudo iptables -F
```

### Reading the output

Each progress line looks like:

```
progress: 10.0 s, 287.4 tps, lat 68.2 ms stddev 12.4, 0 failed
```

| pgbench column | Notes |
|---|---|
| `tps` | instantaneous throughput; drops to 0 during partition / leader election |
| `lat` | mean latency; spikes when tablet leaders are re-elected |
| `stddev` | spikes sharply during node failure |
| `failed` | failed transactions that second |

On AWS, expect lower `tps` and higher `lat` than local — real network latency between EC2 instances adds round-trip cost to every Raft consensus round. Recovery time after the partition heals depends on Raft election timeouts over the real WAN RTT.