# Memcached Extstore Performance Benchmarking

## Contents

- [Overview & Architecture](#overview--architecture)
- [System Environment](#system-environment)
- [Server Setup (`husky10`)](#server-setup-husky10)
- [Client Setup (`husky09`)](#client-setup-husky09)
- [System Bottleneck Analysis & Learnings](#system-bottleneck-analysis--learnings)
- [Summary of Learnings](#summary-of-learnings)
- [Future Work](#future-work)

## Overview & Architecture

This repository documents the bottleneck analysis and key learnings from benchmarking Memcached with Extstore as part of my URA with Prof. Martin Karsten. Future work to be done is also outlined at the bottom.

---

## System Environment

There are two primary nodes connected via a 10-Gigabit Ethernet (10GbE) link:

| Role       | Hostname  | Hardware Profile    | Storage Media             |
| :--------- | :-------- | :------------------ | :------------------------ |
| **Client** | `husky09` | 12 core x86_64 host | SATA SSD                  |
| **Server** | `husky10` | 64 core x86_64 host | 2x NVMe SSDs (1.5TB each) |

Physical link speeds and network interface controllers (NICs) were verified on the server (`husky10`):

```bash
ip link
sudo ethtool enp2s0f0np0
lspci | grep -i ethernet
```

- **Active Network Interface:** `enp2s0f0np0`
- **Negotiated Link Speed:** `10000Mb/s` (10 Gbps Full Duplex)
- **NIC Hardware:** Intel Corporation Ethernet Controller E810-XXV for SFP (rev 02) & Intel 10-Gigabit X540-AT2 (rev 01)
- **Physical Upper Bound:** Network payload bandwidth is capped at **10 Gbps (~1.25 GB/s theoretical maximum)**.

The NVMe SSD on husky10 was verified to be HWE36P43016M000N MLC (likely with no SLC buffer).

- via `lsblk -d -o NAME,MODEL,SERIAL,SIZE`

---

## Server Setup (`husky10`)

### Package Installation

```bash
sudo apt update && sudo apt install -y memcached netcat-openbsd
```

### NVMe Storage Formatting & Mounting for Extstore

Memcached Extstore offloads large key-value items from DRAM onto non voilative memory. We need to format the unused nvme decide with a filesystem and mount it.

```bash
lsblk
sudo mkfs.ext4 /dev/nvme0n1
sudo mkdir -p /mnt/nvme_storage
sudo mount /dev/nvme0n1 /mnt/nvme_storage
sudo chown -R $USER:$USER /mnt/nvme_storage
```

### Launching Memcached Daemon with Extstore

Launch Memcached as a background daemon configured with a 20 GB Extstore pool on the mounted NVMe drive:

```bash
memcached -u $USER \
  -l <server IP> \
  -p 11211 \
  -m 4096 \
  -t 64 \
  -c 16384 \
  -o ext_path=/mnt/nvme_storage/extstore.data:20G,ext_item_size=8192,ext_threads=4,ext_wbuf_size=2 \
  -d
```

#### Parameters:

- `-l <server IP> -p 11211`: Binds to the designated server IP and port 11211.
- `-m 4096`: Allocates 4 GB of RAM for in memory store.
- `-t 64`: Spawns 64 worker threads to handle incoming client network connections.
- `-c 16384`: Allows up to 16,384 concurrent connection sockets.
- `ext_path=/mnt/nvme_storage/extstore.data:20G`: Allocates a 20 GB file on the fast NVMe mount.
- `ext_item_size=8192`: Threshold size (8 KB). Items larger than 8,192 bytes are automatically offloaded to NVMe, anything below stays in memory.
- `ext_threads=4`: Allocates 4 dedicated asynchronous I/O threads for disk writes/reads.
- `ext_wbuf_size=2`: Assigns a 2 MB write buffer per Extstore thread to coalesce flash writes.

---

## Client Setup (`husky09`)

### Repository Setup & Dependencies

`mutilate` was selected as the benchmarking client. To install, run

```bash
git clone https://github.com/leverich/mutilate
cd mutilate
sudo apt update
sudo apt install -y build-essential scons libevent-dev gengetopt libzmq3-dev
```

### SConscript Python 3 Patching & Compilation

Because standard Ubuntu (e.g., 24.04 Noble) ships with Python 3 while original `mutilate` build scripts utilize Python 2 syntax, the `SConscript` file was updated:

1. Replaced all legacy Python 2 `print` statements with standard Python 3 `print(...)` function calls.
2. Executed build and increased open file descriptor limits (allows more connections to be sent out):

```bash
scons
# elevate file descriptor limits for high concurrency
ulimit -n 65535
```

### Dataset Warmup & Initial Verification

```bash
./mutilate -s <server IP>:11211 --loadonly
./mutilate -s <server IP>:11211 -T 8 -c 16 -t 10
```

Result:

```
type       avg     std     min     5th    10th    90th    95th    99th
read      456.6   143.6    72.9   275.7   304.3   647.3   709.0   835.5
update      0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0
op_q        1.0     0.0     1.0     1.0     1.0     1.1     1.1     1.1

Total QPS = 280197.0 (2801993 / 10.0s)

Misses = 0 (0.0%)
Skipped TXs = 0 (0.0%)

RX  692572271 bytes :   66.0 MB/s
TX  100872972 bytes :    9.6 MB/s
```

---

## System Bottleneck Analysis & Learnings

### Experiment 1: Network-Bound Test

#### Workload Execution:

```bash
./mutilate -s <server IP>:11211 -T 12 -c 512 --valuesize=6192 -t 30
```

Result:

```
type       avg     std     min     5th    10th    90th    95th    99th
read    33023.6 22312.9  2726.4 12421.8 15291.8 60894.6 68997.9 96863.9
update      0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0
op_q        1.0     0.0     1.0     1.0     1.0     1.1     1.1     1.1

Total QPS = 186015.3 (5582960 / 30.0s)

Misses = 0 (0.0%)
Skipped TXs = 0 (0.0%)

RX 34838393797 bytes : 1107.0 MB/s
TX  200990196 bytes :    6.4 MB/s
```

#### Server Telemetry Metrics:

1. **Network Interface (`sar -n DEV 1`):**
   - Interface `enp2s0f0np0` reached **98.21% `%ifutil`**.
   - Transmit rate: **1,198,909 kB/s (~1.19 GB/s)**.
2. **Disk I/O (`iostat -xz 1`):**
   - NVMe utilization (`%util`) was **0.00%**.
   - `%iowait` across CPU cores was **0.00%**.
3. **CPU Utilization (`mpstat` / `top`):**
   - User CPU: `2.26%`
   - System CPU (Kernel TCP/IP processing): `19.62%`
   - Idle CPU: `78.12%`

#### Analysis:

Values sized at $6,192B$ were below the `ext_item_size=8192` offloading threshold, serving all queries directly out of memory. The workload hit a physical bottleneck at the 10GbE network interface\*\* (98.21% interface utilization), while storage remained untouched and a large amount of CPU capacity remained unused. So we can conclude that this workload was network bound.

---

### Experiment 2: Storage Bound Extstore Test (with Linux Page Cache)

To force data offloading onto the NVMe Extstore layer, server DRAM was intentionally restricted to 32 MB and populated with large items.

#### Server Execution:

```bash
pkill memcached
memcached -u $USER \
  -l <server IP> \
  -p 11211 \
  -m 32 \
  -t 64 \
  -c 16384 \
  -o ext_path=/mnt/nvme_storage/extstore.data:20G,ext_item_size=8192,ext_threads=4,ext_wbuf_size=2 \
  -d
```

#### Workload Execution:

```bash
# load
./mutilate -s <server IP>:11211 --loadonly --records=1000000 --valuesize=10000
# run benchmark
./mutilate -s <server IP>:11211 --records=1000000 --valuesize=10000 -T 12 -c 512 -t 60
```

Result:

```
type       avg     std     min     5th    10th    90th    95th    99th
read    19058.6  8904.6  7071.6 11813.8 12240.7 28977.4 31751.7 55153.6
update      0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0
op_q        1.0     0.0     1.0     1.0     1.0     1.1     1.1     1.1

Total QPS = 322240.6 (19342837 / 60.0s)

Misses = 17452392 (90.2%)
Skipped TXs = 0 (0.0%)

RX 19121318429 bytes :  303.8 MB/s
TX  696342132 bytes :   11.1 MB/s
```

#### System Behavior:

Initially, the results looked exactly the same as our network bound test, but after a while we get:

`iostat -xz 1`:

- NVMe Write Operations (w/s): ~15,993.00 w/s
- NVMe Write Bandwidth (wkB/s): ~2,047,104 kB/s (~2.05 GB/s)
- I/O Queue & Await Latency: Queue size (aqu-sz) escalated to 228.60 – 249.49, with write await (w_await) at 14.29 – 15.60 ms.
- Disk Utilization (%util): 98.20% – 100.00% on /dev/nvme0n1.

### Analysis

With only 32 MB of DRAM and 10KB item sizes, all of the items should get offloaded to extstore (SSD), so why are the results initially the same? The reason is probably because of the Linux page cache. Extstore in this experiment uses a 2MB write buffer in memory before sending items to a write syscall. The linux page cache then caches these writes in memory, which makes writing to the SSD seem really quick during most of the benchmark. However, once the OS Page Cache filled up completely, the system flushed these writes into the SSD, causing the storage bottleneck observed by iostat above.

---

## Experiment 3: Direct I/O (`O_DIRECT`) Storage-Bound Test

To bypass kernel page cache, Memcached was modified to use `O_DIRECT` for for Extstore storage reads/writes (see memcached folder of this repository for changes). Linux file alignment for O_DIRECT require file offsets, lengths, and memory buffers to be aligned to filesystem block boundaries (4096 bytes).

Memcached was built, compiled and tested via the commands in the README of [github.com/memcached/memcached](https://github.com/memcached/memcached)

Server launch command:

```
./memcached -u yf4zhang \
  -l <server IP> \
  -p 11211 \
  -m 32 \
  -t 64 \
  -c 16384 \
  -o ext_path=/mnt/nvme_storage/extstore.data:20G,ext_item_size=8192,ext_threads=4,ext_wbuf_size=2 \
  -d
```

Workload:
`./mutilate -s <server IP>:11211 --records=1000000 --valuesize=10000 -T 12 -c 512 -t 60`

Result:

```
type       avg     std     min     5th    10th    90th    95th    99th
read     2015.3  3432.5    66.3   130.0   145.1  7817.4  8229.4  8590.3
update      0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0
op_q        1.0     0.0     1.0     1.0     1.0     1.1     1.1     1.1

Total QPS = 101591.4 (6095490 / 60.0s)

Misses = 4717926 (77.4%)
Skipped TXs = 0 (0.0%)

RX 13929463417 bytes :  221.4 MB/s
TX  219658392 bytes :    3.5 MB/s
```

### Analysis:

In the first 40 seconds, the results were network bound, with usage of SSD this time:

- Network (`sar -n DEV 1`): rxkB/s = 1,190,063.95 (~1.19 GB/s), %ifutil = 97.49%

- Disk (`iostat -xz 1`): Writes = 2,960 w/s (~378.8 MB/s), %util = 13.0% - 14.2%

However, in the last ~20 seconds of the benchmark, results seemed to be completely storage bound

- Network (`sar -n DEV 1`): Transmit payload drops to txkB/s = 243,117.35 (~243 MB/s), %ifutil = 19.92%

- Disk (`iostat -xz 1`): Direct reads reach 22,936 r/s (~317.2 MB/s), driving %util to 99.9% – 100.0%

We can see from these results that overall throughput decrease to 221 MB/s, which makes sense since we are bypassing the kernel cache. We are also definitely using more SSD than experiment 2, verifying that the O_DIRECT flag has worked. I suspect the reason why storage jumps so high in the last few seconds may be of two reasons:

1. Extstore page compaction activates later on to reclaim fragmented page, adding internal read/write overhead on the SSD, shooting its usage up to 100%

2. Our SSD has a SLC buffer (SLC is faster than MLC) that makes initial reads/writes super fast (this is unlikely with our type of SSD, since our SSD is enterprise).

## Summary of Learnings

1. There is a clear shift between network to storage bottleneck between the experiments due to the item size change.

2. The linux page cache can cause issues while benchmarking storage bound workloads, we need a way to get around the page cache (`O_DIRECT`)

3. Page compaction might be driving up SSD usage later onto the benchmark more than the actual workload.

---

## Future Work

Future work requires more analysis into the exact reason why we get the results we do in experiment 3.
