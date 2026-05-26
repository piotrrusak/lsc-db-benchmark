# ==============================================================================
# COCKROACHDB BENCHMARKING RUNBOOK (FINAL COMPREHENSIVE SCRIPT)
# ==============================================================================

# ------------------------------------------------------------------------------
# PHASE 1: CLIENT MACHINE ENVIRONMENT SETUP (Execute on Client: 10.0.0.14)
# ------------------------------------------------------------------------------
# Update repositories and install the PostgreSQL 16 tool bundle (contains pgbench)
sudo apt update && sudo apt install -y postgresql-16

# Stop and disable the local database daemon to preserve t3.medium RAM
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Verify the wrapper link is fully operational
pgbench --version

# Create the custom, strict Serializable transaction file
cat << 'EOF' > ~/serializable_txn.sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
\set aid random(1, 100000 * :scale)
\set delta random(-5000, 5000)
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
END;
EOF


# ------------------------------------------------------------------------------
# PHASE 2: COCKROACHDB ENGINE INSTALLATION (Execute on ALL Nodes: 10.0.0.11, .12, .13)
# ------------------------------------------------------------------------------
# Fetch and extract the last unthrottled major version (v23.1.26)
curl https://binaries.cockroachdb.com/cockroach-v23.1.26.linux-amd64.tgz | tar -xz

# Link the executable binary globally into the system execution path
sudo cp cockroach-v23.1.26.linux-amd64/cockroach /usr/local/bin/


# ------------------------------------------------------------------------------
# PHASE 3: SECURE CLUSTER STARTUP SEQUENCE
# ------------------------------------------------------------------------------

# [EXECUTE ON NODE 1 - 10.0.0.11]
cockroach start --insecure --store=node1-data --listen-addr=10.0.0.11:26257 --http-addr=10.0.0.11:8080 --join=10.0.0.11:26257,10.0.0.12:26257,10.0.0.13:26257 --cache=1GB --max-sql-memory=1GB --background

# [EXECUTE ON NODE 2 - 10.0.0.12]
cockroach start --insecure --store=node2-data --listen-addr=10.0.0.12:26257 --http-addr=10.0.0.12:8080 --join=10.0.0.11:26257,10.0.0.12:26257,10.0.0.13:26257 --cache=1GB --max-sql-memory=1GB --background

# [EXECUTE ON NODE 3 - 10.0.0.13]
cockroach start --insecure --store=node3-data --listen-addr=10.0.0.13:26257 --http-addr=10.0.0.13:8080 --join=10.0.0.11:26257,10.0.0.12:26257,10.0.0.13:26257 --cache=1GB --max-sql-memory=1GB --background

# [EXECUTE ON CLIENT - 10.0.0.14] Trigger cluster consensus initialization
cockroach init --insecure --host=10.0.0.11:26257


# ------------------------------------------------------------------------------
# PHASE 4: RUN BENCHMARK 1 - THE NORMAL BASELINE (Execute on Client: 10.0.0.14)
# ------------------------------------------------------------------------------
# Provision the target database space
cockroach sql --insecure --host=10.0.0.11 --port=26257 -e "CREATE DATABASE benchmark;"

# Generate 1,000,000 baseline rows safely using Client-Side data generation (-steps=dtgpf)
pgbench -i -s 10 -h 10.0.0.11 -p 26257 -U root -d benchmark --init-steps=dtgpf

# Execute the baseline test run and pipe execution logs to a file
PGOPTIONS="-c client_min_messages=warning" pgbench -c 8 -j 4 -T 120 -P 1 -f ~/serializable_txn.sql -h 10.0.0.11 -p 26257 -U root benchmark > crdb_normal_results.log 2>&1


# ------------------------------------------------------------------------------
# PHASE 5: PURGE AND RESET CLUSTER STATE (Execute on Client: 10.0.0.14)
# ------------------------------------------------------------------------------
# Drop data tables and rebuild blank database containers
cockroach sql --insecure --host=10.0.0.11 --port=26257 -e "DROP DATABASE benchmark; CREATE DATABASE benchmark;"

# Re-generate schema and fresh rows
pgbench -i -s 10 -h 10.0.0.11 -p 26257 -U root -d benchmark --init-steps=dtgpf


# ------------------------------------------------------------------------------
# PHASE 6: RUN BENCHMARK 2 - THE NETWORK PARTITION CHAOS TEST
# ------------------------------------------------------------------------------

# [EXECUTE ON CLIENT - 10.0.0.14] Start Workload Engine
# Launches the transaction client in the background while streaming outputs to terminal and log file simultaneously
PGOPTIONS="-c client_min_messages=warning" pgbench -c 8 -j 4 -T 120 -P 1 -f ~/serializable_txn.sql -h 10.0.0.11 -p 26257 -U root benchmark 2>&1 | tee crdb_partition_results.log &

# [EXECUTE ON NODE 3 - 10.0.0.13 AT SECOND 30] Inject Partition Failure
# Run these lines exactly 30 seconds after the pgbench workload starts
sudo iptables -A INPUT -s 10.0.0.11 -j DROP
sudo iptables -A INPUT -s 10.0.0.12 -j DROP
sudo iptables -A OUTPUT -d 10.0.0.11 -j DROP
sudo iptables -A OUTPUT -d 10.0.0.12 -j DROP

# [EXECUTE ON NODE 3 - 10.0.0.13 AT SECOND 60] Heal Network Partition
# Run this line exactly 60 seconds after the pgbench workload starts to re-join the cluster
sudo iptables -F


# ------------------------------------------------------------------------------
# PHASE 7: CLUSTER TEAR DOWN (Execute locally on Node 1, Node 2, and Node 3)
# ------------------------------------------------------------------------------
# Cleanly disconnect and terminate engine threads
cockroach quit --insecure --host=localhost:26257

# Clear raw data directories from storage volumes
rm -rf node*-data
