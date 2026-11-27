# Building a Two-Node Slurm Cluster on Ubuntu 24.04 from Scratch

Setting up Slurm by hand is one of the best ways to understand how an HPC environment fits together: authentication, scheduling, and job accounting. In this guide we build a small but fully usable cluster with a head node and a compute node.

## What we are building

| Component | Runs on | Purpose |
|---|---|---|
| `slurmctld` | headnode | Slurm controller (scheduler) |
| `slurmdbd` | headnode | Accounting daemon |
| MariaDB | headnode | Database backing job accounting |
| `slurmd` | computenode-01 | Compute daemon that runs jobs |
| Munge | both nodes | Authentication between nodes |

```
                 +------------------+                +--------------------+
                 |     headnode     |                |   computenode-01   |
                 |                  |   Munge auth   |                    |
                 |  slurmctld       |<-------------->|  slurmd            |
                 |  slurmdbd        |                |                    |
                 |  MariaDB         |                |                    |
                 +------------------+                +--------------------+
```

By the end you will be able to submit jobs with `srun` and `sbatch` and have them recorded in the accounting database.

## Prerequisites

- Two Ubuntu 24.04 servers (VMs, EC2 instances, or bare metal)
- `sudo`/root access on both
- Private network connectivity between the two nodes

---

## Step 1: Update and prepare both nodes

Run on **both** nodes:

```bash
sudo apt update -y && sudo apt upgrade -y
```

## Step 2: Set hostnames

```bash
# On the head node
sudo hostnamectl set-hostname headnode

# On the compute node
sudo hostnamectl set-hostname computenode-01
```

## Step 3: Edit `/etc/hosts` on both nodes

Add the following, replacing the placeholders with your private IPs:

```text
<HEADNODE_IP>      headnode
<COMPUTENODE_IP>   computenode-01
```

## Step 4: Create a local user on the compute node

This user is used to copy files (such as the Munge key) between nodes:

```bash
sudo adduser ssh-user
```

---

## Step 5: Install and configure Munge

Munge provides authentication so Slurm daemons on different nodes can trust each other.

On **both** nodes:

```bash
sudo apt install munge libmunge-dev -y
sudo systemctl enable --now munge
```

Verify it works:

```bash
munge -n | unmunge
```

You should see `STATUS: Success (0)`.

### Copy the Munge key from head node to compute node

Every node must share the **same** key. The key is only readable by the `munge` user, so copy it to a readable temporary location first.

```bash
# On the head node
sudo cp /etc/munge/munge.key /tmp/munge.key
sudo chmod 644 /tmp/munge.key
scp /tmp/munge.key ssh-user@computenode-01:/tmp/
sudo rm /tmp/munge.key
```

```bash
# On the compute node
sudo mv /tmp/munge.key /etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl restart munge
```

Confirm cross-node authentication from the head node:

```bash
munge -n | ssh ssh-user@computenode-01 unmunge
```

---

## Step 6: Install MariaDB for job accounting

Perform these steps on the **head node**.

```bash
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

`mysql_secure_installation` is interactive. Read each prompt, set a strong root password, and remember it.

### Create the accounting database and user

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Step 7: Install and configure `slurmdbd`

```bash
sudo apt install slurmdbd -y
```

Create `/etc/slurm/slurmdbd.conf`:

```ini
AuthType=auth/munge
DbdAddr=localhost
DbdHost=localhost
SlurmUser=slurm
DebugLevel=verbose
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageUser=slurm
StoragePass=your_secure_password
```

`slurmdbd` refuses to start if this file is readable by others, because it contains a password. Lock it down:

```bash
sudo chown slurm:slurm /etc/slurm/slurmdbd.conf
sudo chmod 600 /etc/slurm/slurmdbd.conf
```

Start the service:

```bash
sudo systemctl enable --now slurmdbd
sudo systemctl status slurmdbd
```

---

## Step 8: Install and configure the Slurm controller

On the **head node**:

```bash
sudo apt install slurmctld -y
```

Create `/etc/slurm/slurm.conf`:

```ini
ClusterName=ng-ubuntu
SlurmctldHost=headnode
AuthType=auth/munge
ProctrackType=proctrack/cgroup
SchedulerType=sched/backfill
SelectType=select/cons_tres

SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd

SlurmctldPidFile=/var/run/slurm/slurmctld.pid
SlurmdPidFile=/var/run/slurm/slurmd.pid

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=headnode
AccountingStorageUser=slurm

NodeName=computenode-01 CPUs=2 RealMemory=3836 Sockets=1 CoresPerSocket=1 ThreadsPerCore=2 State=UNKNOWN
PartitionName=ng-ubuntu-01 Nodes=ALL Default=YES MaxTime=120 State=UP
```

> **Tip:** Adjust `ClusterName`, `NodeName` hardware values, and `PartitionName` to match your environment. On the compute node, run `slurmd -C` to print the exact `CPUs`, `Sockets`, `CoresPerSocket`, `ThreadsPerCore`, and `RealMemory` values to use.

Start the controller:

```bash
sudo systemctl enable --now slurmctld
sudo systemctl status slurmctld
```

---

## Step 9: Install and configure `slurmd` on the compute node

Copy the **same** `slurm.conf` to the compute node (for example via `scp` to `/tmp`, then `sudo mv` it to `/etc/slurm/slurm.conf`). Then:

```bash
sudo apt install slurmd libpmix-dev -y
sudo systemctl enable --now slurmd
sudo systemctl status slurmd
```

---

## Step 10: Test the cluster

From the head node, check node status:

```bash
sinfo
scontrol show nodes
```

If the node is stuck in `DRAIN` or `UNKNOWN`:

```bash
sudo scontrol update NodeName=computenode-01 State=RESUME
```

### Test with `srun`

```bash
srun hostname
```

This should print `computenode-01`, confirming the job ran on the compute node.

### Test with `sbatch`

```bash
cat <<'EOF' > test.sh
#!/bin/bash
#SBATCH --job-name=test
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --output=test_%j.out
#SBATCH --error=test_%j.err

echo "Running on $(hostname)"
sleep 30
echo "Done"
EOF

sbatch test.sh
squeue
```

When the job finishes, read the output file:

```bash
cat test_*.out
```

> **Note:** Output files are written on the compute node's filesystem. Without shared storage the file lives on `computenode-01`. See the next steps below for NFS.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `munge -n \| unmunge` fails | Is `munge` running? Do both nodes have the identical key with correct ownership and `400` permissions? |
| `slurmdbd` won't start | Permissions on `slurmdbd.conf` (must be `600`, owned by `slurm`), DB credentials, `/var/log/slurm/slurmdbd.log` |
| Node shows `down` or `unk` | Is `slurmd` running? Do both nodes have the same `slurm.conf`? Are hostnames resolvable in `/etc/hosts`? |
| Node stuck in `DRAIN` | `scontrol show node computenode-01` and read the `Reason` field, then `scontrol update ... State=RESUME` |
| Clocks differ | Munge credentials expire, so keep both nodes in sync with `chrony` or `systemd-timesyncd` |
| Connection refused between nodes | Open ports `6817` (slurmctld) and `6818` (slurmd) in your firewall or cloud security group |

Useful logs:

```bash
sudo journalctl -u slurmctld -u slurmdbd -u slurmd --no-pager -n 50
sudo tail -f /var/log/slurm/slurmctld.log
```

---

## Final thoughts

Setting up Slurm manually shows how the pieces of an HPC environment connect: Munge for authentication, `slurmctld` and `slurmd` for scheduling and execution, and `slurmdbd` with MariaDB for accounting. This two-node cluster is a solid foundation for exploring more advanced topics:

- **GPU scheduling** with GRES
- **Containerized workloads** with Apptainer or Enroot/Pyxis
- **Shared storage** such as NFS or Lustre
- **Monitoring** with Prometheus and Grafana
- **Accounting and fair-share** with `sacctmgr` and `sacct`

If this helped you, feel free to star the repo and share your own setup experiences.

---

`#slurm` `#hpc` `#ubuntu` `#linux` `#devops` `#cluster` `#mariadb` `#munge`
