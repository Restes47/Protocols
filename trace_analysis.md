# Wireshark/Tshark Trace Analysis Cheatsheet

**Focus:** Linux, NetApp E-Series, StorageGRID, ONTAP, S3, NFS, iSCSI, and NVMe/TCP

---

## Table of Contents

1. [Tshark Fundamentals](#tshark-fundamentals)
2. [Capture Optimization](#capture-optimization)
3. [Protocol-Specific Analysis](#protocol-specific-analysis)
4. [Performance & Troubleshooting](#performance--troubleshooting)
5. [Linux Command Patterns](#linux-command-patterns)
6. [Quick Reference Filters](#quick-reference-filters)

---

## Tshark Fundamentals

### Installation & Verification

```bash
# Ubuntu/Debian
sudo apt-get install tshark

# Verify installation
tshark -v

# Add user to wireshark group (avoid sudo)
sudo usermod -aG wireshark $USER
newgrp wireshark
```

### Capture vs. Read Workflow

```bash
# Capture directly (preferred for high-volume)
tshark -i eth0 -w output.pcap

# Read existing capture
tshark -r input.pcap

# Display with minimal overhead
tshark -r input.pcap -Y "filter" -c 1000
```

### Core Tshark Options

| Option | Purpose |
|--------|---------|
| `-i <interface>` | Capture interface (eth0, ens0, any) |
| `-w <file>` | Write capture to file |
| `-r <file>` | Read from file |
| `-f <cfilter>` | Capture filter (BPF syntax, applied at kernel) |
| `-Y <dfilter>` | Display filter (Wireshark syntax, post-capture) |
| `-c <count>` | Stop after N packets |
| `-N` | Disable name resolution |
| `-K <key_log_file>` | Load SSL/TLS keys for decryption |
| `-V` | Verbose output (show all fields) |
| `-e <field>` | Extract specific fields |
| `-T <format>` | Output format (text, json, csv, psml) |
| `-a duration:<sec>` | Auto-stop after N seconds |
| `-b duration:300` | Ring buffer (rotate every 300s) |

---

## Capture Optimization

### High-Speed Captures

```bash
# Disable name resolution (massive speed boost)
tshark -i eth0 -N -w output.pcap

# Capture without packet inspection (kernel-level filtering)
tshark -i eth0 -f "host 192.168.1.100" -w output.pcap

# Ring buffer for continuous capture
tshark -i eth0 -b duration:600 -b files:10 -w capture_

# Snapshot length (reduce filesize, storage traffic often needs full packet)
tshark -i eth0 -s 0 -w output.pcap  # -s 0 = full packet
```

### Targeted Capture Filters (Kernel-Level)

```bash
# iSCSI (port 3260)
tshark -i eth0 -f "tcp port 3260" -w iscsi.pcap

# NFS (ports 111, 2049)
tshark -i eth0 -f "tcp port 2049 or udp port 111 or udp port 2049" -w nfs.pcap

# NVMe/TCP (port 4420)
tshark -i eth0 -f "tcp port 4420" -w nvme_tcp.pcap

# S3 (HTTP/HTTPS, often port 80/443 but can be custom)
tshark -i eth0 -f "tcp port 80 or tcp port 443" -w s3.pcap

# ONTAP: iSCSI, NFS, S3, NDMP
tshark -i eth0 -f "tcp port 3260 or tcp port 2049 or tcp port 443 or tcp port 10000" -w ontap.pcap

# E-Series: iSCSI, Ethernet (specify target IPs)
tshark -i eth0 -f "tcp port 3260 and (host 192.168.1.50 or host 192.168.1.51)" -w eseries.pcap

# StorageGRID: S3, Swift (often 443, 8082, 8084)
tshark -i eth0 -f "tcp port 443 or tcp port 8082 or tcp port 8084" -w storagegrid.pcap

# Multi-protocol (combine with parentheses)
tshark -i eth0 -f "(tcp port 3260) or (tcp port 2049) or (tcp port 4420)" -w storage.pcap
```

---

## Protocol-Specific Analysis

### iSCSI

#### Capture & Analysis

```bash
# Capture iSCSI traffic
tshark -i eth0 -f "tcp port 3260" -w iscsi.pcap

# View iSCSI PDU types
tshark -r iscsi.pcap -Y "iscsi" -e frame.number -e iscsi.opcode -e iscsi.initiator_task_tag -T fields

# Extract iSCSI session details
tshark -r iscsi.pcap -Y "iscsi.login" -V | grep -i "initiator name\|target name\|session id"

# Find iSCSI CDB (command descriptor block)
tshark -r iscsi.pcap -Y "iscsi.scsi_cdb" -V

# iSCSI errors (check for failed commands)
tshark -r iscsi.pcap -Y "iscsi.response_status != 0" -e frame.number -e iscsi.response_status -T fields
```

#### Key Fields to Monitor

```
iscsi.opcode                 # PDU type (0=NOP-Out, 1=SCSI, 2=TMF, etc.)
iscsi.initiator_task_tag     # Matches request to response
iscsi.stat                   # Status (0=success)
iscsi.scsi_cdb               # SCSI command bytes
iscsi.data_length            # I/O size
iscsi.flags                  # F (final), C (continue)
```

#### Performance Analysis

```bash
# Measure iSCSI I/O latency
tshark -r iscsi.pcap -Y "iscsi.scsi_command" -e frame.time -e iscsi.initiator_task_tag -T fields > requests.txt
tshark -r iscsi.pcap -Y "iscsi.scsi_response" -e frame.time -e iscsi.initiator_task_tag -T fields > responses.txt
# Cross-reference timestamps to calculate latency

# Find long I/O operations
tshark -r iscsi.pcap -Y "iscsi.scsi_response and iscsi.response_status == 0" -e frame.number -e iscsi.data_length -T fields | awk -F',' '$2 > 1000000 {print}'
```

#### Troubleshooting

```bash
# Login failures
tshark -r iscsi.pcap -Y "iscsi.login and iscsi.login_status != 0" -V

# Connection drops (RST packets)
tshark -r iscsi.pcap -Y "tcp.flags.reset" -e frame.time -e ip.src -e ip.dst -T fields

# Task aborts
tshark -r iscsi.pcap -Y "iscsi.function == 6" -V  # TMF abort task

# PDU errors
tshark -r iscsi.pcap -Y "iscsi.stat != 0 or iscsi.status != 0" -V
```

---

### NFS (v3 & v4)

#### Capture & Analysis

```bash
# Capture NFS (both UDP and TCP)
tshark -i eth0 -f "tcp port 2049 or udp port 2049 or udp port 111" -w nfs.pcap

# Show NFS procedures
tshark -r nfs.pcap -Y "nfs" -e frame.number -e nfs.procedure_v3 -e nfs.status -T fields

# NFS v4 operations
tshark -r nfs.pcap -Y "nfs.opcode" -e frame.number -e nfs.opcode -e nfs.nfsstat4 -T fields

# Find NFS reads
tshark -r nfs.pcap -Y "nfs.procedure_v3 == 7" -e frame.number -e nfs.read.count -T fields

# Find NFS writes
tshark -r nfs.pcap -Y "nfs.procedure_v3 == 8" -e frame.number -e nfs.write.count -T fields
```

#### Key Fields to Monitor

```
nfs.procedure_v3                # NFS operation (0=NULL, 1=GETATTR, 3=LOOKUP, 6=READ, 7=WRITE, etc.)
nfs.status                      # NFS status (0=OK, 3=NOACCESS, 5=IO, etc.)
nfs.read.count / nfs.write.count # I/O size
nfs.fh.length                   # File handle length
nfs.name                        # Filename/path
```

#### Performance Analysis

```bash
# Measure NFS operation latency
tshark -r nfs.pcap -Y "nfs.call_id" -e frame.time -e nfs.call_id -e nfs.procedure_v3 -T fields > calls.txt
# Correlate call IDs between request/response

# Slow reads/writes
tshark -r nfs.pcap -Y "nfs.procedure_v3 == 6" -e frame.number -e nfs.read.count -T fields | awk '$2 < 4096 {print}'

# NFS errors
tshark -r nfs.pcap -Y "nfs.status != 0" -e frame.number -e nfs.status -e nfs.procedure_v3 -T fields
```

#### Troubleshooting

```bash
# Port mapper issues (UDP 111)
tshark -r nfs.pcap -Y "rpcap.status != 0" -V

# Permission denied
tshark -r nfs.pcap -Y "nfs.status == 3" -e frame.number -e nfs.name -T fields

# Stale filehandles
tshark -r nfs.pcap -Y "nfs.status == 70" -V

# Fragmented NFS packets
tshark -r nfs.pcap -Y "ip.flags.mf == 1 or ip.frag_offset > 0" -e frame.number -e ip.len -T fields
```

---

### iSCSI vs. NFS Performance Comparison

```bash
# Extract timing data from both protocols
tshark -r mixed.pcap -Y "iscsi.scsi_command" -e frame.time_delta -T fields | awk '{sum+=$1; count++} END {print "iSCSI avg delta: " sum/count}'

tshark -r mixed.pcap -Y "nfs.procedure_v3 == 6" -e frame.time_delta -T fields | awk '{sum+=$1; count++} END {print "NFS avg delta: " sum/count}'
```

---

### NVMe/TCP

#### Capture & Analysis

```bash
# Capture NVMe/TCP
tshark -i eth0 -f "tcp port 4420" -w nvme_tcp.pcap

# Show NVMe/TCP PDU types
tshark -r nvme_tcp.pcap -Y "nvme_tcp" -e frame.number -e nvme_tcp.hdr.type -T fields

# Extract command info
tshark -r nvme_tcp.pcap -Y "nvme_tcp.hdr.type == 0" -V | grep -i "opcode\|command id\|nsid"

# Find NVMe completions
tshark -r nvme_tcp.pcap -Y "nvme_tcp.hdr.type == 1" -e frame.number -e nvme_tcp.cqe.status -T fields
```

#### Key Fields to Monitor

```
nvme_tcp.hdr.type              # 0=CapsulCmd, 1=CapsulResp, 2=H2CData, 3=C2HData
nvme_tcp.cmd.opcode            # Command opcode (0=Read, 1=Write, etc.)
nvme_tcp.cmd.command_id         # Correlate request/response
nvme_tcp.cqe.status             # Completion status
nvme_tcp.data_length            # Transfer length
```

#### Performance Analysis

```bash
# NVMe command latency
tshark -r nvme_tcp.pcap -Y "nvme_tcp.hdr.type == 0" -e frame.number -e frame.time -e nvme_tcp.cmd.command_id -T fields > nvme_cmd.txt

tshark -r nvme_tcp.pcap -Y "nvme_tcp.hdr.type == 1" -e frame.number -e frame.time -e nvme_tcp.cqe.command_id -T fields > nvme_cpl.txt

# Correlate command IDs to calculate latency

# High bandwidth operations
tshark -r nvme_tcp.pcap -Y "nvme_tcp" -e frame.number -e nvme_tcp.data_length -T fields | awk '$2 > 65536 {print}'
```

---

### S3 (HTTP/HTTPS)

#### Capture & Analysis

```bash
# Capture S3 traffic (typically HTTPS on 443 or custom port)
tshark -i eth0 -f "tcp port 443 or tcp port 8082 or tcp port 9000" -w s3.pcap

# Decode TLS (requires keylog file)
tshark -r s3.pcap -K /path/to/keylog.txt -w s3_decrypted.pcap

# Extract HTTP methods and paths
tshark -r s3_decrypted.pcap -Y "http" -e frame.number -e http.request.method -e http.request.uri -T fields

# S3 PUT operations (uploads)
tshark -r s3_decrypted.pcap -Y "http.request.method == PUT" -e frame.number -e http.request.uri -e http.content_length -T fields

# S3 GET operations (downloads)
tshark -r s3_decrypted.pcap -Y "http.request.method == GET" -e frame.number -e http.request.uri -T fields

# S3 response codes
tshark -r s3_decrypted.pcap -Y "http.response" -e frame.number -e http.response.code -e http.response.phrase -T fields
```

#### Key Fields to Monitor

```
http.request.method             # GET, PUT, POST, DELETE, HEAD
http.request.uri                # S3 bucket/object path
http.content_length             # Upload/download size
http.response.code              # 200=OK, 404=NotFound, 503=ServiceUnavailable, etc.
ssl.handshake.type              # TLS handshake phase
```

#### StorageGRID-Specific

```bash
# StorageGRID S3 on port 8082 (HTTP) or 8084 (HTTPS)
tshark -i eth0 -f "tcp port 8082 or tcp port 8084" -w storagegrid_s3.pcap

# Extract S3 auth headers
tshark -r storagegrid_s3.pcap -Y "http" -V | grep -i "authorization\|host\|date"

# Monitor multipart uploads
tshark -r storagegrid_s3.pcap -Y "http.request.uri contains \"uploadId\"" -e frame.number -e http.request.method -T fields
```

#### Performance Analysis

```bash
# Measure S3 PUT latency
tshark -r s3.pcap -Y "http.request.method == PUT" -e frame.time -e http.request.uri -T fields > s3_puts.txt
tshark -r s3.pcap -Y "http.response.code == 200" -e frame.time -e http.request.uri -T fields > s3_responses.txt

# Calculate throughput (bytes per second)
tshark -r s3.pcap -Y "http.response" -e frame.time -e http.content_length -T fields | awk '{bytes+=$2; print $1, bytes}'
```

#### Troubleshooting

```bash
# TLS errors (handshake failures)
tshark -r s3.pcap -Y "tls.alert" -V

# HTTP errors (4xx, 5xx)
tshark -r s3.pcap -Y "http.response.code >= 400" -e frame.number -e http.response.code -e http.response.phrase -T fields

# Slow requests (timeout detection)
tshark -r s3.pcap -Y "tcp.flags.reset or tcp.connection_reset" -e frame.time -e ip.src -e ip.dst -T fields
```

---

### ONTAP Protocol Analysis

#### Multi-Protocol Monitoring

```bash
# Capture all common ONTAP protocols
tshark -i eth0 -f "(tcp port 443) or (tcp port 3260) or (tcp port 2049) or (tcp port 4420) or (tcp port 10000)" -w ontap.pcap

# Separate by protocol in post-analysis
tshark -r ontap.pcap -Y "iscsi" -w ontap_iscsi.pcap
tshark -r ontap.pcap -Y "nfs" -w ontap_nfs.pcap
tshark -r ontap.pcap -Y "tcp.port == 443" -w ontap_s3.pcap
```

#### ONTAP-Specific Analysis

```bash
# NDMP (backup) traffic on port 10000
tshark -i eth0 -f "tcp port 10000" -w ontap_ndmp.pcap

# CIFS/SMB (port 445)
tshark -i eth0 -f "tcp port 445" -w ontap_smb.pcap

# SNAPMIRROR (async replication)
tshark -r ontap.pcap -Y "tcp.dstport == 20048 or tcp.dstport == 20049" -V

# Cluster communication (ICMP, TCP 11104-11105)
tshark -i eth0 -f "tcp port 11104 or tcp port 11105 or icmp" -w ontap_cluster.pcap
```

#### Performance Tuning Analysis

```bash
# Measure snapshot transfer speed
tshark -r ontap.pcap -Y "iscsi.data_length > 0" -e frame.time -e iscsi.data_length -T fields | awk '{sum+=$2; count++} END {print "Total bytes: " sum ", Count: " count}'

# Identify bottlenecks (high latency operations)
tshark -r ontap.pcap -Y "tcp.flags.ack" -e frame.time_delta -T fields | awk '$1 > 0.1 {print "Slow ack: " $1}'
```

---

### E-Series Analysis

#### E-Series Controller Communication

```bash
# Capture E-Series iSCSI data (specify controller IPs)
tshark -i eth0 -f "tcp port 3260 and (host 192.168.1.50 or host 192.168.1.51)" -w eseries_iscsi.pcap

# Monitor E-Series management traffic (HTTP/HTTPS on port 8080/8443)
tshark -i eth0 -f "tcp port 8080 or tcp port 8443" -w eseries_mgmt.pcap

# Dual-controller failover detection (connection resets)
tshark -r eseries_iscsi.pcap -Y "tcp.flags.reset" -e frame.time -e ip.src -e ip.dst -T fields

# E-Series cache behavior (frequent reads from same LBA)
tshark -r eseries_iscsi.pcap -Y "iscsi.scsi_cdb" -e frame.number -e iscsi.scsi_cdb -T fields | grep -i "lba"
```

#### E-Series Performance Metrics

```bash
# Average I/O size
tshark -r eseries_iscsi.pcap -Y "iscsi.data_length > 0" -e iscsi.data_length -T fields | awk '{sum+=$1; count++} END {print "Avg I/O: " sum/count " bytes"}'

# Queue depth (outstanding commands)
tshark -r eseries_iscsi.pcap -Y "iscsi.scsi_command" -e iscsi.initiator_task_tag -T fields | sort | uniq | wc -l
```

---

## Performance & Troubleshooting

### General Latency Analysis

```bash
# Extract all frame timestamps and calculate statistics
tshark -r capture.pcap -T fields -e frame.time_relative | awk 'NR>1 {sum+=$1; sumsq+=$1*$1; count++} END {avg=sum/count; var=(sumsq/count)-(avg*avg); print "Avg: " avg ", StdDev: " sqrt(var)}'

# Identify outliers (frames with unusual delays)
tshark -r capture.pcap -T fields -e frame.time_delta -e frame.number | awk '$1 > 0.05 {print "Frame " $2 ": " $1 " sec delay"}'
```

### Network Conditions

```bash
# Detect packet loss (sequence number gaps in TCP)
tshark -r capture.pcap -Y "tcp" -e tcp.seq -e tcp.len -T fields | awk 'NR>1 {if ($2 != prev+$3 && prev > 0) print "Gap detected"; prev=$2}'

# Identify retransmissions
tshark -r capture.pcap -Y "tcp.analysis.retransmission" -e frame.number -e tcp.seq -T fields

# Monitor duplicate ACKs (congestion indicator)
tshark -r capture.pcap -Y "tcp.analysis.duplicate_ack" -e frame.number -e tcp.ack -T fields | head -20

# Packet fragmentation
tshark -r capture.pcap -Y "ip.flags.mf == 1 or ip.frag_offset > 0" -e frame.number -e ip.len -T fields
```

### Connection Quality

```bash
# TCP window size evolution (throughput limiting factor)
tshark -r capture.pcap -Y "tcp.flags.ack" -e frame.number -e tcp.window_size -T fields | tail -50

# MTU path discovery issues (ICMP fragmentation needed)
tshark -r capture.pcap -Y "icmp.type == 3 and icmp.code == 4" -V | grep -i "mtu"

# Connection resets and timeouts
tshark -r capture.pcap -Y "tcp.flags.reset or tcp.flags.fin" -e frame.time -e ip.src -e ip.dst -e tcp.flags -T fields
```

### Protocol-Level Issues

```bash
# Timeout detection (long idle periods on live capture)
tshark -i eth0 -N -f "tcp port 2049" -Y "tcp.flags.fin or tcp.flags.reset" | grep -i "time="

# Half-open connections (SYN-ACK but no data)
tshark -r capture.pcap -Y "tcp.flags.syn and tcp.flags.ack" -e ip.src -e ip.dst -T fields | sort | uniq -c | awk '$1 > 5 {print}'

# Protocol state mismatches
tshark -r capture.pcap -Y "tcp.flags.psh or tcp.flags.urg" -e frame.number -e tcp.flags -T fields
```

---

## Linux Command Patterns

### Integration with tcpdump

```bash
# Live capture with tcpdump, analyze with tshark
sudo tcpdump -i eth0 -w - host 192.168.1.100 | tshark -r - -Y "tcp.flags.fin" -e frame.number -e ip.src -T fields

# Capture only on specific port, auto-stop after 100MB
sudo tcpdump -i eth0 -w capture.pcap -C 100 tcp port 3260

# Real-time analysis with dumpcaps
sudo -g wireshark tcpdump -i eth0 -w - | tshark -r - -a duration:60 -w result.pcap
```

### File Processing & Analysis

```bash
# Merge multiple capture files
mergecap -w combined.pcap file1.pcap file2.pcap file3.pcap

# Extract specific conversations
tshark -r capture.pcap -Y "ip.src == 192.168.1.100 and ip.dst == 192.168.1.200" -w conversation.pcap

# Split capture into manageable chunks
tshark -r large.pcap -b files:1000 -w split_

# Convert to JSON for programmatic analysis
tshark -r capture.pcap -T json -Y "tcp.port == 3260" > analysis.json
```

### Statistics & Reporting

```bash
# Generate summary statistics
tshark -r capture.pcap -q -z io,stat,1

# Top talkers
tshark -r capture.pcap -q -z ip_hosts,tree

# Protocol distribution
tshark -r capture.pcap -q -z protocol,tree

# TCP stream analysis
tshark -r capture.pcap -q -z tcp,follow -e tcp.stream

# Endpoint statistics
tshark -r capture.pcap -q -z endpoints,tcp

# Frame size distribution
tshark -r capture.pcap -e frame.len -T fields | awk '{sum+=$1; count++; hist[int($1/1000)]++} END {for (i in hist) print "Frames " i "KB: " hist[i]; print "Avg: " sum/count}'
```

### Automated Troubleshooting

```bash
# Alert on high-latency I/O
tshark -r capture.pcap -Y "iscsi.data_length > 0" -e frame.time_delta -T fields | awk '$1 > 0.5 {print "SLOW I/O: " $1 " sec"}'

# Alert on errors
tshark -r capture.pcap -Y "iscsi.stat != 0 or nfs.status != 0 or http.response.code >= 400" -e frame.number -e ip.src -e ip.dst -T fields > errors.txt

# Generate report for specific timeframe
tshark -r capture.pcap -Y "frame.time > \"2024-09-27 14:00:00\" and frame.time < \"2024-09-27 15:00:00\"" -w hourly_report.pcap

# Export to CSV for Excel analysis
tshark -r capture.pcap -Y "iscsi" -e frame.number -e frame.time -e ip.src -e ip.dst -e iscsi.data_length -T csv > iscsi_analysis.csv
```

### Real-Time Monitoring

```bash
# Monitor iSCSI performance live (top 10 talkers)
tshark -i eth0 -f "tcp port 3260" -Y "iscsi.data_length > 0" -e ip.src -e iscsi.data_length -T fields | awk '{src_bytes[$1]+=$2} END {for (i in src_bytes) print src_bytes[i], i}' | sort -rn | head -10

# Live NFS latency monitoring
tshark -i eth0 -f "tcp port 2049" -Y "nfs.procedure_v3 == 6" -e frame.time_delta -T fields | awk '{if ($1 > 0.05) print "SLOW NFS READ: " $1 " sec"}'

# Watch for connection failures
tshark -i eth0 -f "tcp" -Y "tcp.flags.reset or tcp.flags.fin" -e frame.time -e ip.src -e ip.dst | while read line; do echo "[ALERT] Connection closed: $line"; done
```

---

## Quick Reference Filters

### iSCSI Filters

```
iscsi.scsi_command              # Sent by initiator
iscsi.scsi_response             # Sent by target
iscsi.login                     # Login requests/responses
iscsi.logout                    # Logout
iscsi.data_out                  # Write data
iscsi.data_in                   # Read data
iscsi.r2t                       # Ready to Transfer
iscsi.nop_out                   # NOP (keepalive)
iscsi.stat != 0                 # Errors
iscsi.initiator_task_tag        # Track individual I/Os
iscsi.data_length > 1048576     # Large I/Os (>1MB)
```

### NFS Filters

```
nfs.procedure_v3 == 0           # NULL
nfs.procedure_v3 == 1           # GETATTR
nfs.procedure_v3 == 2           # SETATTR
nfs.procedure_v3 == 3           # LOOKUP
nfs.procedure_v3 == 4           # ACCESS
nfs.procedure_v3 == 5           # READLINK
nfs.procedure_v3 == 6           # READ
nfs.procedure_v3 == 7           # WRITE
nfs.procedure_v3 == 8           # CREATE
nfs.procedure_v3 == 9           # MKDIR
nfs.procedure_v3 == 10          # SYMLINK
nfs.procedure_v3 == 11          # MKNOD
nfs.procedure_v3 == 12          # REMOVE
nfs.procedure_v3 == 13          # RMDIR
nfs.procedure_v3 == 14          # RENAME
nfs.procedure_v3 == 15          # LINK
nfs.procedure_v3 == 16          # READDIR
nfs.procedure_v3 == 17          # READDIRPLUS
nfs.procedure_v3 == 18          # FSSTAT
nfs.procedure_v3 == 19          # FSINFO
nfs.procedure_v3 == 20          # PATHCONF
nfs.procedure_v3 == 21          # COMMIT
nfs.status != 0                 # Errors
nfs.read.count                  # Read size
nfs.write.count                 # Write size
```

### NVMe/TCP Filters

```
nvme_tcp.hdr.type == 0          # Command capsule
nvme_tcp.hdr.type == 1          # Response capsule
nvme_tcp.hdr.type == 2          # H2CData (Host to Controller)
nvme_tcp.hdr.type == 3          # C2HData (Controller to Host)
nvme_tcp.cmd.opcode == 0        # Read command
nvme_tcp.cmd.opcode == 1        # Write command
nvme_tcp.cmd.opcode == 2        # Flush
nvme_tcp.cqe.status != 0        # Errors
nvme_tcp.data_length > 65536    # Large transfers
```

### S3/HTTP Filters

```
http.request.method == "GET"    # S3 read
http.request.method == "PUT"    # S3 write
http.request.method == "DELETE" # S3 delete
http.request.method == "HEAD"   # S3 metadata
http.response.code == 200       # Success
http.response.code == 404       # Not found
http.response.code == 503       # Service unavailable
http.response.code >= 400       # All errors
http.request.uri contains "uploadId" # Multipart upload
ssl.handshake                   # TLS negotiation
tls.alert                       # TLS errors
```

### TCP/Network Filters

```
tcp.flags.syn                   # Connection start
tcp.flags.fin                   # Connection end
tcp.flags.reset                 # Connection reset
tcp.flags.ack                   # Acknowledgment
tcp.analysis.retransmission     # Packet resent
tcp.analysis.duplicate_ack      # Duplicate ACK
tcp.analysis.fast_retransmission # Fast retransmit
ip.flags.mf == 1                # Fragmented packets
ip.frag_offset > 0              # Fragment reassembly
icmp.type == 3 and icmp.code == 4 # ICMP frag needed
tcp.window_size == 0            # Zero window (flow control)
```

### Combined Filters

```
# iSCSI reads/writes only
(iscsi.scsi_command or iscsi.scsi_response) and iscsi.data_length > 0

# NFS with errors
nfs and nfs.status != 0

# S3 multipart uploads from specific host
http and ip.src == 192.168.1.100 and http.request.uri contains "uploadId"

# Storage traffic with retransmissions
(iscsi or nfs or nvme_tcp) and tcp.analysis.retransmission

# High-latency iSCSI I/Os
iscsi.scsi_response and iscsi.data_length > 262144

# Mixed protocol analysis
(iscsi or nfs or nvme_tcp) and (tcp.flags.reset or tcp.analysis.retransmission)
```

---

## Pro Tips for Faster Analysis

### Capture Optimization
1. **Use capture filters** (`-f`) to reduce file size at kernel level—avoid processing gigabytes
2. **Disable DNS resolution** (`-N`)—massive speed boost on large captures
3. **Specify snapshot length** (`-s 0` for full packets vs. `-s 64` for headers only)
4. **Use ring buffers** (`-b duration:600 -b files:10`) for continuous monitoring

### Analysis Optimization
1. **Load only what you need**: Use `-Y` (display filter) to process post-capture
2. **Extract to fields** (`-e field -T fields`) for piping to awk/grep
3. **Use JSON output** for programmatic analysis and version control
4. **Index by time range**: Pre-filter by frame.time to narrow dataset

### Common Workflow
```bash
# Step 1: Capture (optimized)
sudo tshark -i eth0 -N -f "tcp port 3260" -w iscsi_raw.pcap -a duration:300

# Step 2: Initial analysis (high-level)
tshark -r iscsi_raw.pcap -q -z io,stat,10

# Step 3: Deep dive (filtered)
tshark -r iscsi_raw.pcap -Y "iscsi.stat != 0" -V > errors.txt

# Step 4: Export for Excel/Python
tshark -r iscsi_raw.pcap -Y "iscsi.data_length > 0" -e frame.number -e frame.time -e iscsi.initiator_task_tag -e iscsi.data_length -T csv > analysis.csv
```

### Regex & Pattern Matching
```bash
# Find slow operations
tshark -r capture.pcap -e frame.time_delta -T fields | awk '$1 ~ /^0\.[5-9]|^[1-9]/ {print "Slow: " $1}'

# Extract IP pairs
tshark -r capture.pcap -Y "tcp" -e ip.src -e ip.dst -T fields | sort | uniq -c | sort -rn | head -10

# Find patterns in URIs
tshark -r s3.pcap -Y "http" -e http.request.uri -T fields | grep -oE "(bucket|key|uploadId)=[^ &]+" | sort | uniq -c | sort -rn
```

---

## Key Resources

- **Wireshark Wiki**: https://wiki.wireshark.org/
- **Protocol Dissectors**: https://wiki.wireshark.org/Dissectors
- **Display Filter Reference**: https://www.wireshark.org/docs/dfref/
- **Capture Filter Syntax**: https://wiki.wireshark.org/CaptureFilters
- **NetApp Knowledge Base**: https://kb.netapp.com/
- **iSCSI RFC 3720**: https://tools.ietf.org/html/rfc3720
- **NFS RFC 1813** (v3): https://tools.ietf.org/html/rfc1813
- **NVMe/TCP RFC 7540**: https://tools.ietf.org/html/rfc7540

---

**Last Updated**: 2026-09-27  
**Version**: 1.0  
**Created for Linux/NetApp/StorageGRID analysis**
