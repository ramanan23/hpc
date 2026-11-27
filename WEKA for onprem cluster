# Deploying WEKA Parallel Storage for an On-Prem HPC Cluster (with Slurm Integration)

A scheduler and compute nodes are only half of an HPC cluster. Without fast shared storage, GPUs and CPUs sit idle waiting for I/O. This post walks through designing and deploying a **WEKA** parallel file system on bare metal and integrating it with a **Slurm** cluster, with a focus on the technical details that matter: network design, core and NUMA planning, protection schemes, client mounts, and CPU pinning.

> **Version note:** WEKA is commercial software and CLI syntax and requirements change between releases. Commands marked *illustrative* are examples of the workflow, not a copy-paste runbook. Always validate against the documentation for your version: <https://docs.weka.io>

---

## 1. Architecture overview

WEKA is a distributed, parallel, NVMe-optimized file system. It has two roles:

| Role | What it is | Runs on |
|---|---|---|
| **Backend** | Servers with NVMe drives that store data and serve I/O | Dedicated storage servers |
| **Client** | Hosts that mount the filesystem via the `wekafs` driver | Slurm compute nodes, login nodes, head node |

```
  +-------------------------------------------------------------+
  |  Slurm head node        Login nodes        Compute nodes    |
  |  (WEKA client)          (WEKA client)      (WEKA clients)   |
  +--------------------------+----------------------------------+
                             |  Storage fabric (Ethernet / InfiniBand)
                             |  MTU >= 4k, DPDK-capable NICs
  +--------------------------+----------------------------------+
  |  WEKA backend: 8+ servers, each with NVMe drives,           |
  |  dedicated cores per WEKA process, 2+ NICs                  |
  +-------------------------------------------------------------+
```

Key design properties:

- **Kernel-bypass I/O.** WEKA uses DPDK for high-performance networking, which is why NIC choice and dedicated CPU cores matter.
- **Distributed metadata and data.** Files are striped across backend servers with erasure-coded protection, so there is no single metadata server bottleneck.
- **Shared namespace.** Every node mounts the same filesystem, so job inputs, outputs, and checkpoints are visible everywhere.

---

## 2. Capacity and hardware planning

### 2.1 Minimum cluster size

The WEKA 5.0 prerequisites document lists a minimal configuration of **8 servers** for a new on-prem cluster (6 for cloud). Older vendor reference architectures describe 6-server building blocks with 4+2 protection, so check the requirements for the version you are deploying.

### 2.2 Protection scheme (stripe width and parity)

Data protection is expressed as `N+P`, where `N` is data units and `P` is parity units.

| Scheme | Notes |
|---|---|
| 4+2 | Typical for small clusters, higher capacity overhead |
| 8+2 | Lower overhead, recommended when a larger cluster is expected |
| N+4 | Only available on clusters with 14 or more servers (per Lenovo's reference design) |

Choose the scheme **before** deployment based on expected growth. Changing stripe geometry later is not trivial.

### 2.3 Server specification checklist

| Component | Guidance |
|---|---|
| **Drives** | High-performance NVMe only. Pick endurance (1 DWPD vs 3 DWPD) based on write intensity. Keep drive counts balanced across NUMA nodes. |
| **CPU** | Favor high single-thread performance and enough cores to dedicate to WEKA processes. Cores dedicated to WEKA are not available to other workloads. |
| **Memory** | Size per WEKA's memory planning formulas for your version and container layout. Reference designs use 256 to 512 GB per node. |
| **Boot drive** | NVMe SSD, 960 GB minimum per the 5.0 prerequisites. |
| **NICs** | At least 2 high-speed adapters supported by WEKA (for example ConnectX-6 class at up to 200 Gb/s). A third adapter can be dedicated to object-tier or multi-protocol traffic. |
| **Time sync** | All servers synchronized (NTP/chrony). |
| **Watchdog** | A watchdog driver at `/dev/watchdog` (hardware watchdog recommended). |
| **`/opt/weka`** | Critical to WEKA operation. Keep it on local storage, or on highly available shared storage if you must. |

### 2.4 Core and NUMA mapping

This is the part most people get wrong. WEKA runs multiple process types per container (drives, compute, frontend), each pinned to dedicated cores. Reference designs often deploy **multiple WEKA containers per server** (for example two containers with 15 cores each) and **map core IDs and NVMe drives to the same NUMA node**, so I/O never crosses the socket interconnect.

Inspect your topology before planning:

```bash
lscpu | grep -i numa
lstopo-no-graphics --no-io        # hwloc, shows cores per NUMA node
lspci -nn | grep -i nvme
for d in /sys/class/nvme/nvme*; do echo "$d -> NUMA $(cat $d/device/numa_node)"; done
for n in /sys/class/net/*/device/numa_node; do echo "$n: $(cat $n)"; done
```

Rule of thumb: for each NUMA node, WEKA cores, NVMe drives, and the NIC serving them should all live on that node.

---

## 3. Network design

The storage fabric is the heart of the system.

### 3.1 Requirements

- **MTU:** WEKA recommends setting MTU to **at least 4k** on the NICs of WEKA servers and the connected switches.
- **Jumbo frames rule:** if any network connection on a backend can transmit frames larger than 4 KB, then every connection WEKA uses directly on that backend must handle at least 4 KB.
- **IOMMU:** WEKA detects and enables IOMMU automatically for the server and PCI devices, so you do not need to enable it manually.
- **Separation:** keep WEKA traffic on its own fabric or VLAN, separate from your Slurm management and user networks.

### 3.2 Switch and host settings (Ethernet example)

```bash
# Host side (illustrative)
sudo ip link set dev ens1f0 mtu 9000

# Verify end-to-end MTU with the Don't Fragment bit
ping -M do -s 8972 <backend-ip>
```

On Ethernet fabrics, configure lossless behavior (PFC/ECN) on the switches according to your NIC and switch vendor guidance. On InfiniBand, ensure the subnet manager is healthy and the partition and MTU settings are consistent.

### 3.3 IP planning

Allocate a dedicated IP range for WEKA. Each WEKA container (drives, compute, frontend) typically needs its own IP on the storage network, so plan more IPs than servers. Keep a spreadsheet of container-to-IP-to-NIC-to-NUMA mappings.

---

## 4. Choosing an installation path

On bare metal, WEKA documents several paths:

| Path | When to use |
|---|---|
| **WMS with WSA** | Preferred and fastest path on supported Dell, HPE, Lenovo, and Supermicro servers |
| **WSA** (WEKA Software Appliance) | Speeds up OS deployment and WEKA install when WMS prerequisites cannot be met |
| **WEKA Configurator** | Generates a cluster configuration script (`config.sh`) to apply on the servers |
| **Manual (Resources Generator)** | Different OS, customized images, or unsupported hardware. Requires deep knowledge of WEKA architecture |

WSA and the installation packages are obtained through get.weka.io and WEKA's customer success team. After WSA installs, servers boot into **STEM mode**, ready for configuration.

---

## 5. Deployment workflow

### Step 1: Prepare hosts

On every backend server:

```bash
# Time sync
sudo timedatectl set-ntp true
chronyc tracking

# Confirm watchdog device
ls -l /dev/watchdog

# Confirm NVMe drives are visible and unused
lsblk -d -o NAME,SIZE,MODEL,TRAN | grep nvme
sudo nvme list

# Confirm hugepages / kernel settings per WEKA's host-preparation guide
```

Also complete the NIC configuration (MTU, IPs, routes, SR-IOV if you use virtual functions) from WEKA's "Prepare the system" documentation.

### Step 2: Install the WEKA software

Obtain the installation package from get.weka.io, then install it on each backend server per your chosen path.

### Step 3: Generate and apply the cluster configuration

With the WEKA Configurator, you describe the servers, drives, NICs, and core allocation, and it produces a `config.sh` that creates the containers and forms the cluster. A multi-container backend typically consists of three container types per server:

| Container | Runs | Purpose |
|---|---|---|
| `drives0` | Drive processes | Owns the NVMe drives |
| `compute0` | Compute processes | Filesystem logic, erasure coding, metadata |
| `frontend0` | Frontend processes | Serves client and protocol traffic |

*Illustrative* container creation (the Configurator generates the exact commands for you):

```bash
weka local setup container --name drives0 --base-port 14000 \
    --cores 2 --only-drives-cores --net ens1f0/<ip>/<prefix>/<gw>

weka local setup container --name compute0 --base-port 14300 \
    --cores 4 --only-compute-cores --net ens1f0/<ip>/<prefix>/<gw>

weka local setup container --name frontend0 --base-port 14200 \
    --cores 1 --only-frontend-cores --net ens1f0/<ip>/<prefix>/<gw>
```

Then form the cluster and add drives *(illustrative)*:

```bash
weka cluster create <host1> <host2> ... <host8>
weka cluster drive add <container-id> /dev/nvme0n1 /dev/nvme1n1 ...
weka cluster update --cluster-name hpc-weka
weka cluster start-io
```

### Step 4: Apply the license and verify

```bash
weka cluster license payload      # get payload to request a license
weka cluster license set <license>
weka status
weka cluster container
weka cluster drive
weka alerts
```

Wait for the cluster to report **OK** with all containers `UP` and no active rebuilds.

### Step 5: Create filesystems

*Illustrative:*

```bash
weka fs group create default
weka fs create home default 20TiB
weka fs create scratch default 150TiB
weka fs create projects default 80TiB
```

Plan separate filesystems per purpose so you can apply different quotas, snapshot policies, and (if licensed) tiering:

| Filesystem | Use | Suggested policy |
|---|---|---|
| `home` | User home directories | Quotas, daily snapshots |
| `scratch` | Job temp and I/O-heavy data | No snapshots, purge policy |
| `projects` | Shared project data | Quotas per project, snapshots |

Licensing note: the XPS edition covers the flash tier inside the WEKA servers, and the Data Tiering Option (DTO) is an add-on needed if you tier data to an object backend.

---

## 6. Client setup on Slurm nodes

### 6.1 Install the WEKA client

Run on the head node, login nodes, and every compute node *(illustrative)*:

```bash
curl http://<backend-ip>:14000/dist/v1/install | sudo sh
```

### 6.2 Mount in DPDK mode with dedicated cores

For best performance, the client runs in DPDK mode with dedicated cores. The number of client cores is the most significant mount option: Lenovo's reference design notes that 1 or 2 dedicated cores is the typical choice, and 2 is generally a good one depending on client network bandwidth and workload I/O.

```bash
sudo mkdir -p /mnt/weka
sudo mount -t wekafs -o num_cores=2,net=ens1f0 <backend-ip>/scratch /mnt/weka/scratch
```

Persistent mount in `/etc/fstab` *(illustrative)*:

```text
<backend-ip>/scratch  /mnt/weka/scratch  wekafs  num_cores=2,net=ens1f0,_netdev  0 0
```

Trade-off: cores given to the WEKA client are **no longer available to compute jobs**. Budget for this when you size compute nodes.

### 6.3 Mount everywhere, consistently

Use the **same mount path on all nodes** so job scripts work unchanged wherever Slurm places them:

```bash
srun -N <num_nodes> df -h /mnt/weka/scratch
srun -N <num_nodes> bash -c 'echo $(hostname): $(ls /mnt/weka/scratch | head -1)'
```

---

## 7. Slurm integration: CPU pinning

If WEKA clients use dedicated cores, Slurm must not schedule jobs onto them. Two changes are required, based on Lenovo's reference architecture guidance.

**1. Reserve the cores in Slurm.** Use core specialization on the node definition so those cores are excluded from job cgroups and from `slurmd` use:

```ini
# /etc/slurm/slurm.conf
NodeName=computenode-[01-16] CPUs=64 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 \
    RealMemory=256000 CoreSpecCount=2 State=UNKNOWN
```

If you need to choose specific cores, use `CpuSpecList` instead, and make sure it matches the cores WEKA actually uses on that node.

**2. Tell WEKA not to isolate cpusets.** In the client configuration, set `isolate_cpusets` to `false`:

```bash
# /etc/wekaio/weka.conf
isolate_cpusets=false
```

Restart the client and `slurmd`, then verify:

```bash
scontrol show node computenode-01 | grep -E 'CPUTot|CoreSpec|CPUSpecList'
weka local ps                                  # container health
weka local resources                           # cores assigned to the client
taskset -cp $(pgrep -f weka | head -1)         # which cores WEKA is bound to
```

Also match **NUMA locality**: dedicate WEKA cores on the socket attached to the storage NIC.

Add `/mnt/weka` health to your node health check so a node with a failed mount is drained rather than running jobs that fail:

```bash
# /etc/slurm/healthcheck.sh (HealthCheckProgram)
mountpoint -q /mnt/weka/scratch || scontrol update nodename=$(hostname -s) state=drain reason="weka mount missing"
```

---

## 8. Validation and benchmarking

Never put production workloads on the cluster before measuring it.

### 8.1 Single-client throughput with `fio`

```bash
# Sequential read, large blocks
fio --name=seqread --directory=/mnt/weka/scratch --rw=read --bs=1M \
    --size=20G --numjobs=8 --iodepth=32 --direct=1 --ioengine=libaio \
    --group_reporting

# Random 4k read IOPS
fio --name=randread --directory=/mnt/weka/scratch --rw=randread --bs=4k \
    --size=10G --numjobs=16 --iodepth=64 --direct=1 --ioengine=libaio \
    --group_reporting
```

### 8.2 Multi-node parallel I/O with IOR and mdtest

```bash
module load mpi ior
srun -N 8 --ntasks-per-node=16 ior -a POSIX -b 4g -t 1m -F -e -g -o /mnt/weka/scratch/ior_test
srun -N 8 --ntasks-per-node=16 mdtest -n 5000 -d /mnt/weka/scratch/mdtest -u
```

Watch for:

- Aggregate bandwidth scaling with client count until the fabric or backend saturates
- Metadata ops/sec from `mdtest`, which stresses the small-file path
- Latency tails (p99) during mixed workloads

Vendor reference designs publish headline figures for fully loaded clusters (for example 180 GB/s read and 14M read IOPS on a Lenovo reference), but your results depend on drives, network, node count, and protection scheme.

---

## 9. Operations

### Monitoring

```bash
weka status
weka alerts
weka stats --interval 10
weka cluster rebuild-status     # name may vary by version
```

Export metrics to Prometheus and visualize them in Grafana. At minimum, alert on:

- Container or drive down
- Rebuild in progress
- Capacity above 80 percent
- Network errors or MTU mismatches
- Client mount failures

### Safe server shutdown

WEKA provides a systemd-based safe-shutdown mechanism. Always use the documented procedure before rebooting or powering down a backend server, and never power off multiple servers at once beyond your protection level.

### Upgrades

Plan client and backend upgrades together. Client versions need to be compatible with the backend, so coordinate client upgrades with the Slurm maintenance window. Drain nodes first:

```bash
scontrol update nodename=computenode-[01-16] state=drain reason="weka client upgrade"
```

### Capacity and growth

You scale by adding backend servers. Adding servers increases both capacity and performance. Remember the protection-scheme decision from section 2.2, since it influences how you grow.

---

## 10. Troubleshooting cheat sheet

| Symptom | Likely cause | Check |
|---|---|---|
| Mount hangs or fails | Wrong network or port, firewall, MTU mismatch | `ping -M do -s <size>`, `weka local ps`, backend port 14000 reachable |
| Low throughput | Too few client cores, NUMA misalignment, MTU mismatch | `num_cores`, NIC NUMA node, switch MTU |
| Jobs slow when client is busy | Slurm jobs competing with WEKA cores | `CoreSpecCount`, `isolate_cpusets=false`, taskset output |
| Node shows many errors after reboot | Client container not started before jobs | `weka local ps`, systemd ordering, health check |
| Drive or container down | Hardware fault or process failure | `weka cluster drive`, `weka cluster container`, `weka alerts` |
| Time skew warnings | NTP not synced | `chronyc tracking` on all backends |

---

## 11. Summary

Putting WEKA behind a Slurm cluster is mostly a design exercise:

1. **Size the backend properly:** 8+ servers on the 5.0 docs, NVMe-only, balanced across NUMA nodes.
2. **Treat the network as a first-class component:** dedicated fabric, MTU of at least 4k, consistent end-to-end.
3. **Decide on protection and growth early:** 4+2 vs 8+2 vs N+4.
4. **Use DPDK client mounts** with 1 to 2 dedicated cores per compute node.
5. **Tell Slurm about those cores** with `CoreSpecCount` or `CpuSpecList`, and set `isolate_cpusets=false` on the WEKA client.
6. **Benchmark before go-live** with `fio`, IOR, and mdtest, then monitor continuously.

Next steps worth exploring: tiering to an object store, multi-protocol access (NFS, SMB, S3), Kubernetes CSI integration, quotas and per-project filesystems, and disaster recovery with snapshots and replication.

---

## References

- WEKA bare-metal planning and installation: <https://docs.weka.io/planning-and-installation>
- WEKA prerequisites and compatibility (5.0): <https://docs.weka.io/5.0/planning-and-installation/prerequisites-and-compatibility>
- Lenovo EveryScale design architecture for WEKA: <https://lenovopress.lenovo.com/lp1700-lenovo-everyscale-design-architecture-for-weka-storage>
- Lenovo HPFS solution with WEKA: <https://lenovopress.lenovo.com/lp1691>

`#weka` `#hpc` `#slurm` `#storage` `#nvme` `#dpdk` `#linux` `#devops`
