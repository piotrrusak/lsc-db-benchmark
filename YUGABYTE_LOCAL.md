# YugabyteDB Local Benchmark

## Prerequisites

Download and extract YugabyteDB:

```bash
wget https://downloads.yugabyte.com/releases/2.20.7.0/yugabyte-2.20.7.0-b12-linux-x86_64.tar.gz
tar xvfz yugabyte-2.20.7.0-b12-linux-x86_64.tar.gz
cd yugabyte-2.20.7.0/
```

Install Python dependencies required by YugabyteDB post-install script:

```bash
./bin/post_install.sh
```

## Set Up Loopback Aliases

YugabyteDB nodes on a single machine require distinct IP addresses. Add two loopback aliases:

```bash
sudo ip addr add 127.0.0.2/8 dev lo
sudo ip addr add 127.0.0.3/8 dev lo
```

Verify:

```bash
ip addr show lo
```

## Start the 3-Node Cluster

Start node 1:

```bash
./bin/yugabyted start \
  --advertise_address=127.0.0.1 \
  --base_dir=$HOME/yugabyte-data/node1 \
  --cloud_location=local.local.rack1
```

Start node 2:

```bash
./bin/yugabyted start \
  --advertise_address=127.0.0.2 \
  --base_dir=$HOME/yugabyte-data/node2 \
  --join=127.0.0.1 \
  --cloud_location=local.local.rack2
```

Start node 3:

```bash
./bin/yugabyted start \
  --advertise_address=127.0.0.3 \
  --base_dir=$HOME/yugabyte-data/node3 \
  --join=127.0.0.1 \
  --cloud_location=local.local.rack3
```

## Verify Cluster Status

```bash
./bin/yugabyted status --base_dir=$HOME/yugabyte-data/node1
```

Check all nodes are joined:

```bash
./bin/ysqlsh -h 127.0.0.1 -c "SELECT host, port, node_type, cloud, region, zone, public_ip FROM yb_servers();"
```

Expected output:

```
    host    | port | node_type | cloud  | region | zone  | public_ip
------------+------+-----------+--------+--------+-------+----------
 127.0.0.1  | 5433 | primary   | local  | local  | rack1 | 127.0.0.1
 127.0.0.2  | 5433 | primary   | local  | local  | rack2 | 127.0.0.2
 127.0.0.3  | 5433 | primary   | local  | local  | rack3 | 127.0.0.3
(3 rows)
```

## Initialize the Bank Workload

YugabyteDB is PostgreSQL-compatible — use `pgbench` to simulate a bank transfer workload (equivalent to CockroachDB's `bank` workload).

Create the database and initialize pgbench schema (1000 accounts by default with `-s 1`):

```bash
./bin/ysqlsh -h 127.0.0.1 -c "CREATE DATABASE bank;"
pgbench -i -h 127.0.0.1 -p 5433 -U yugabyte -d bank -s 10
```

Verify tables were created:

```bash
./bin/ysqlsh -h 127.0.0.1 -d bank -c "\dt"
```

## Baseline Benchmark (All 3 Nodes Healthy)

Run a 2-minute TPC-B benchmark (transfer-like transactions) with 20 concurrent clients:

```bash
pgbench -h 127.0.0.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 120 -P 1
```

- `-c 20` — 20 concurrent clients
- `-j 4` — 4 worker threads
- `-T 120` — run for 120 seconds
- `-P 1` — print progress every 1 second

### Reading the output

`pgbench` output format differs from CockroachDB's `cockroach workload`. Each progress line looks like:

```
progress: 10.0 s, 1287.4 tps, lat 15.2 ms stddev 3.4, 0 failed
progress: 11.0 s, 1304.1 tps, lat 14.8 ms stddev 2.9, 0 failed
```

| pgbench column | CockroachDB equivalent | Notes |
|---|---|---|
| elapsed time | `_elapsed` | seconds since start |
| `tps` | `ops/sec(inst)` | instantaneous throughput that second |
| `lat` | roughly `p50(ms)` | mean latency, not median |
| `stddev` | — | latency spread; spikes during failures |
| `failed` | `_errors` | failed transactions that second |

There is no real-time p95/p99 column. Tail latency spikes during node failure are visible as `stddev` jumping. The final summary prints overall average latency and total tps.

Example final summary:

```
transaction type: <builtin: TPC-B (sort of)>
number of clients: 20
duration: 120 s
number of transactions actually processed: 154321
number of failed transactions: 0 (0.000%)
latency average = 15.5 ms
latency stddev = 4.1 ms
tps = 1285.9 (without initial connection time)
```

## Node Failure Test — Kill Node 2 During Benchmark

Start a 5-minute benchmark in one terminal:

```bash
pgbench -h 127.0.0.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 300 -P 1
```

In a second terminal, after ~30 seconds kill node 2:

```bash
./bin/yugabyted stop --base_dir=$HOME/yugabyte-data/node2
```

In the benchmark output you will see `tps` drop toward 0 and `stddev` spike as tablet leaders are re-elected. Once quorum stabilizes (2 of 3 nodes are still alive), `tps` recovers:

```
progress: 30.0 s, 1327.3 tps, lat 14.1 ms stddev 3.2, 0 failed
progress: 31.0 s,  441.2 tps, lat 18.9 ms stddev 24.7, 0 failed   <- node killed
progress: 32.0 s,    0.0 tps, lat 0.000 ms stddev 0.000, 0 failed  <- leader election
progress: 33.0 s,    0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
progress: 34.0 s, 1302.1 tps, lat 14.6 ms stddev 3.8, 0 failed    <- recovered
```

Revive node 2 after observing the failure:

```bash
./bin/yugabyted start \
  --advertise_address=127.0.0.2 \
  --base_dir=$HOME/yugabyte-data/node2 \
  --join=127.0.0.1 \
  --cloud_location=local.local.rack2
```

Check all nodes are live again:

```bash
./bin/ysqlsh -h 127.0.0.1 -c "SELECT host, port FROM yb_servers();"
```

## Node Failure Test — Kill Node 3 During Benchmark

Ensure all 3 nodes are healthy, then repeat:

```bash
pgbench -h 127.0.0.1 -p 5433 -U yugabyte -d bank \
  -c 20 -j 4 -T 300 -P 1
```

Kill node 3:

```bash
./bin/yugabyted stop --base_dir=$HOME/yugabyte-data/node3
```

Revive node 3:

```bash
./bin/yugabyted start \
  --advertise_address=127.0.0.3 \
  --base_dir=$HOME/yugabyte-data/node3 \
  --join=127.0.0.1 \
  --cloud_location=local.local.rack3
```

## Benchmark With Error Counting

`pgbench` does not stop on errors by default — unlike CockroachDB's workload which does. Failed transactions are counted in the `failed` column each second and in the final summary. Kill a node and watch:

```
progress: 31.0 s,    0.0 tps, lat 0.000 ms stddev 0.000, 17 failed
progress: 32.0 s,    0.0 tps, lat 0.000 ms stddev 0.000, 42 failed
progress: 33.0 s, 1289.4 tps, lat 14.9 ms stddev 3.3, 0 failed
```

The final summary will show the cumulative error count:

```
number of failed transactions: 59 (0.039%)
```

## Teardown

Stop all nodes and remove data:

```bash
./bin/yugabyted destroy --base_dir=$HOME/yugabyte-data/node1
./bin/yugabyted destroy --base_dir=$HOME/yugabyte-data/node2
./bin/yugabyted destroy --base_dir=$HOME/yugabyte-data/node3
```

Remove loopback aliases:

```bash
sudo ip addr del 127.0.0.2/8 dev lo
sudo ip addr del 127.0.0.3/8 dev lo
```
