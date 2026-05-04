# YugabyteDB AWS Benchmark

## Architecture

Three EC2 instances, each running one YugabyteDB node, deployed in the same VPC for low-latency communication. The benchmark client runs from one of the nodes or a separate instance in the same VPC.

## Step 1 — Launch EC2 Instances

Launch **3 EC2 instances** with the following configuration:

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2023 (or Ubuntu 22.04) |
| Instance type | `t3.medium` or larger |
| VPC | Same VPC for all 3 instances |
| Subnet | Can be same or different subnets in the same region |
| Key pair | Same key pair for all 3 instances |
| Storage | At least 20 GB gp3 |

Note the **private IP addresses** of all three instances after launch (e.g., `172.31.10.1`, `172.31.10.2`, `172.31.10.3`).

## Step 2 — Configure Security Group

Create or modify a security group and add the following **inbound rules**:

| Type | Protocol | Port range | Source | Purpose |
|---|---|---|---|---|
| Custom TCP | TCP | 5433 | Security group (self) | YSQL inter-node |
| Custom TCP | TCP | 7000 | Security group (self) | YB-Master HTTP UI |
| Custom TCP | TCP | 7100 | Security group (self) | YB-Master RPC |
| Custom TCP | TCP | 9000 | Security group (self) | YB-TServer HTTP UI |
| Custom TCP | TCP | 9100 | Security group (self) | YB-TServer RPC |
| Custom TCP | TCP | 15433 | Security group (self) | yugabyted UI |
| Custom TCP | TCP | 5433 | Your IP | YSQL client access |
| Custom TCP | TCP | 7000 | Your IP | Web UI access (optional) |
| SSH | TCP | 22 | Your IP | SSH access |

Apply this security group to all 3 instances.

## Step 3 — Install YugabyteDB on All 3 Instances

SSH into each instance and run the following commands. Repeat on all 3 nodes.

```bash
ssh -i your-key.pem ec2-user@<instance-public-ip>
```

Download and extract YugabyteDB:

```bash
wget https://downloads.yugabyte.com/releases/2.20.7.0/yugabyte-2.20.7.0-b12-linux-x86_64.tar.gz
tar xvfz yugabyte-2.20.7.0-b12-linux-x86_64.tar.gz
cd yugabyte-2.20.7.0/
./bin/post_install.sh
```

## Step 4 — Start the Cluster

Use the **private IP addresses** of the instances throughout. Replace the example IPs below with your actual private IPs.

**On node 1** (`172.31.10.1`):

```bash
./bin/yugabyted start \
  --advertise_address=172.31.10.1 \
  --cloud_location=aws.us-east-1.us-east-1a
```

**On node 2** (`172.31.10.2`):

```bash
./bin/yugabyted start \
  --advertise_address=172.31.10.2 \
  --join=172.31.10.1 \
  --cloud_location=aws.us-east-1.us-east-1b
```

**On node 3** (`172.31.10.3`):

```bash
./bin/yugabyted start \
  --advertise_address=172.31.10.3 \
  --join=172.31.10.1 \
  --cloud_location=aws.us-east-1.us-east-1c
```

## Step 5 — Verify Cluster Status

From any node:

```bash
./bin/yugabyted status --base_dir=$HOME/var
```

Check all nodes are registered:

```bash
./bin/ysqlsh -h 172.31.10.1 -c "SELECT host, port, node_type FROM yb_servers();"
```

Expected output:

```
     host      | port | node_type
---------------+------+-----------
 172.31.10.1   | 5433 | primary
 172.31.10.2   | 5433 | primary
 172.31.10.3   | 5433 | primary
(3 rows)
```

## Step 6 — Install pgbench on the Client Node

Run the benchmark from node 1 (or a separate client instance in the same VPC). Install `pgbench`:

```bash
# Amazon Linux 2023
sudo dnf install -y postgresql15

# Ubuntu
sudo apt-get install -y postgresql-client
```

## Step 7 — Initialize the Bank Workload

Create the database and initialize the pgbench schema:

```bash
./bin/ysqlsh -h 172.31.10.1 -c "CREATE DATABASE bank;"
pgbench -i -h 172.31.10.1 -p 5433 -U yugabyte -d bank -s 10
```

Verify:

```bash
./bin/ysqlsh -h 172.31.10.1 -d bank -c "\dt"
```

## Step 8 — Baseline Benchmark (All 3 Nodes Healthy)

Run a 5-minute TPC-B (bank transfer) benchmark:

```bash
pgbench -h 172.31.10.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 300 -P 1
```

### Reading the output

Each progress line looks like:

```
progress: 10.0 s, 287.4 tps, lat 68.2 ms stddev 12.4, 0 failed
```

| pgbench column | CockroachDB equivalent | Notes |
|---|---|---|
| elapsed time | `_elapsed` | seconds since start |
| `tps` | `ops/sec(inst)` | instantaneous throughput that second |
| `lat` | roughly `p50(ms)` | mean latency — expect higher on AWS due to real network RTT |
| `stddev` | — | spikes sharply during node failure / leader election |
| `failed` | `_errors` | failed transactions that second |

On AWS, expect significantly lower `tps` and higher `lat` than local — real network latency between EC2 instances adds round-trip cost to every Raft consensus round.

Record the steady-state `tps` and `lat` numbers.

## Step 9 — Node Failure Test — Kill Node 2 During Benchmark

Start a 5-minute benchmark on node 1:

```bash
pgbench -h 172.31.10.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 300 -P 1
```

After ~30 seconds, SSH into node 2 and stop it:

```bash
# On node 2
./bin/yugabyted stop
```

Watch the benchmark terminal. `tps` will drop to 0 and `stddev` will spike as tablet leaders are re-elected. On AWS, recovery takes longer than locally because higher inter-node latency slows the Raft election timeout:

```
progress: 30.0 s,  289.1 tps, lat 67.4 ms stddev 11.2, 0 failed
progress: 31.0 s,   91.3 tps, lat 89.2 ms stddev 98.7, 0 failed   <- node killed
progress: 32.0 s,    0.0 tps, lat 0.000 ms stddev 0.000, 0 failed  <- leader election
...
progress: 38.0 s,  271.4 tps, lat 71.3 ms stddev 13.1, 0 failed   <- recovered
```

Revive node 2 after observing the failure:

```bash
# On node 2
./bin/yugabyted start \
  --advertise_address=172.31.10.2 \
  --join=172.31.10.1 \
  --cloud_location=aws.us-east-1.us-east-1b
```

Verify all nodes are live:

```bash
./bin/ysqlsh -h 172.31.10.1 -c "SELECT host, port FROM yb_servers();"
```

## Step 10 — Node Failure Test — Kill Node 3 During Benchmark

Confirm all 3 nodes are healthy, then start the benchmark again:

```bash
pgbench -h 172.31.10.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 300 -P 1
```

SSH into node 3 and stop it:

```bash
# On node 3
./bin/yugabyted stop
```

Revive node 3:

```bash
# On node 3
./bin/yugabyted start \
  --advertise_address=172.31.10.3 \
  --join=172.31.10.1 \
  --cloud_location=aws.us-east-1.us-east-1c
```

## Step 11 — Check Node Status After Recovery

```bash
./bin/ysqlsh -h 172.31.10.1 -c "SELECT host, port, node_type FROM yb_servers();"
```

You can also view the YugabyteDB web UI at `http://<node1-public-ip>:7000` for a visual overview of cluster health, tablet leaders, and replication status.

## Teardown

Stop all nodes to avoid ongoing EC2 charges:

```bash
# On each instance
./bin/yugabyted stop
```

Then terminate the EC2 instances from the AWS console or CLI:

```bash
aws ec2 terminate-instances --instance-ids i-xxxx i-yyyy i-zzzz
```
