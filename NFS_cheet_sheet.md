# NFS Cheat Sheet: NetApp ONTAP, StorageGRID, Linux, and Permissions

## Scope

This guide covers practical NFS configuration and troubleshooting with NetApp Data ONTAP/ONTAP in mind, along with Linux filesystem structure, permissions, and StorageGRID considerations.

> **Important:** ONTAP command syntax can vary by release. Verify commands in the documentation for your ONTAP version before making changes. Treat export-policy and security changes as production-impacting operations.

---

## Quick troubleshooting checklist

1. Verify network connectivity and routing.
2. Verify NFS/RPC services and firewall access.
3. Confirm the ONTAP SVM's NFS service is enabled.
4. Check the export policy and matching client rule.
5. Test a mount with an explicitly selected NFS version and security flavor.
6. If access is denied, compare numeric UID/GID values and check root squashing.
7. Check NFSv4 identity mapping, LDAP/AD, Kerberos, SELinux, and ACLs.
8. For stale handles or locking problems, inspect client messages and ONTAP NFS state.
9. Check ONTAP event logs, network health, and StorageGRID or gateway health where applicable.

---

## NFS fundamentals

### NFS versions

- **NFSv3** is largely stateless and commonly uses RPC services such as `rpcbind`, `mountd`, and lock/statd services in addition to TCP/UDP 2049. Firewalls may need several RPC ports.
- **NFSv4/NFSv4.1** uses TCP 2049 and provides integrated locking, state, ACL support, and identity mapping. It is usually simpler to firewall.

### Security flavors

- `sec=sys`: authentication is based on numeric UNIX UID/GID values.
- `sec=krb5`: Kerberos authentication.
- `sec=krb5i`: Kerberos authentication plus integrity protection.
- `sec=krb5p`: Kerberos authentication plus privacy/encryption.

### Common NFS concepts

- **Client:** Linux host mounting the export.
- **Server:** ONTAP SVM, or another NFS service.
- **Export:** Namespace made available to clients.
- **Mount point:** Local directory where the export is attached.
- **UID/GID:** Numeric user and group identifiers used for UNIX permission checks.
- **Root squashing:** Maps client root to an anonymous user unless explicitly allowed by the server policy.

---

## Linux NFS commands

Replace placeholders such as `<server>`, `<svm>`, and `<mountpoint>` with environment-specific values.

### Network and port checks

```bash
ping -c 4 <server>
traceroute <server>
getent hosts <server>
nc -vz <server> 2049
```

For NFSv3, also check RPC services:

```bash
rpcinfo -p <server>
```

### Discover exports

```bash
showmount -e <server>
showmount -a <server>
```

`showmount` depends on the server exposing the mount protocol and is not a complete NFSv4 diagnostic. An NFSv4 server may be reachable on TCP 2049 even when `showmount` does not return useful information.

### Test mounts explicitly

NFSv4:

```bash
sudo mount -v -t nfs4 -o vers=4 <server>:/<path> /mnt/test
```

NFSv3:

```bash
sudo mount -v -t nfs -o vers=3,proto=tcp <server>:/<export> /mnt/test
```

Test security flavors when configured:

```bash
sudo mount -v -t nfs4 -o vers=4,sec=sys <server>:/<path> /mnt/test
sudo mount -v -t nfs4 -o vers=4,sec=krb5 <server>:/<path> /mnt/test
```

Unmount:

```bash
sudo umount /mnt/test
sudo umount -f /mnt/test   # Use cautiously for an unresponsive mount
sudo umount -l /mnt/test   # Lazy unmount; use cautiously
```

### Inspect mounted filesystems and client state

```bash
findmnt -t nfs,nfs4
mount | grep -E ' type nfs| type nfs4'
cat /proc/mounts | grep -E ' nfs| nfs4'
nfsstat -c
nfsstat -m
```

Kernel and service logs:

```bash
dmesg -T | tail -n 100
journalctl -k --since '15 minutes ago'
journalctl -u nfs-client.target -e
```

Typical messages include `server not responding`, `stale file handle`, authentication errors, and timeout details.

### Check identity and permissions

```bash
id
id <username>
ls -ld /mnt /mnt/test
ls -ln /mnt/test
stat /mnt/test/file
stat -c '%n owner=%u:%g mode=%a (%A)' /mnt/test/file
getfacl /mnt/test/file
```

Check SELinux or AppArmor if a normal UNIX permission check appears correct:

```bash
getenforce
sestatus
ausearch -m AVC -ts recent
aa-status
```

For NFSv4, verify the identity-mapping domain and services. Common locations include `/etc/idmapd.conf` and the distribution's NFS client services.

---

## NetApp ONTAP concepts

- **SVM (Storage Virtual Machine/Vserver):** Logical storage server that provides NFS service and owns the namespace.
- **Volume:** Filesystem container hosted by the SVM.
- **Qtree:** Logical subdivision within a volume that can have its own security and export behavior.
- **Export policy:** Set of rules controlling which clients can access a volume or qtree and whether access is read-only or read-write.
- **Client match:** IP address, subnet, hostname, or other match expression evaluated by the export policy.
- **Superuser setting:** Determines whether client root is treated as root or mapped to the anonymous user.
- **Name services:** Local files, LDAP, NIS, or AD-backed identity sources used to resolve users and groups.

The effective result is determined by more than the export policy: the client must match the policy, the volume must be online, the path must be available, the security flavor must be allowed, and the UNIX/NFSv4 ACLs must grant access.

### ONTAP inspection commands

```text
vserver nfs show -vserver <svm>
vserver nfs status show -vserver <svm>

vserver export-policy show -vserver <svm>
vserver export-policy rule show -vserver <svm> -policyname <policy>

volume show -vserver <svm> -volume <volume>
qtree show -vserver <svm> -volume <volume>

vserver nfs connection show -vserver <svm>
event log show -severity ERROR
volume snapshot show -vserver <svm> -volume <volume>
```

Depending on ONTAP release, additional fields and diagnostic commands may be available for NFS sessions, locks, name services, and Kerberos. Use command completion and `?` in the ONTAP CLI to confirm the exact syntax.

### Export policy items to verify

Check all of the following in `vserver export-policy rule show`:

- The client IP or subnet matches the intended rule.
- The intended protocol (`nfs`, `nfs3`, or `nfs4`, depending on release and configuration) is allowed.
- Read-only versus read-write access is correct.
- The anonymous user ID is appropriate.
- The `superuser` setting is intentional.
- The rule order and overlapping client matches do not produce an unexpected result.

Example rule creation pattern—review carefully before using in production:

```text
vserver export-policy rule create \
    -vserver <svm> \
    -policyname <policy> \
    -clientmatch <client-or-cidr> \
    -rorule sys \
    -rwrule sys \
    -protocol nfs \
    -superuser sys
```

The exact accepted values and options depend on the ONTAP release. Do not grant `superuser` access broadly unless it is required and approved.

---

## StorageGRID considerations

StorageGRID is primarily an object-storage platform exposing S3 and related object APIs; it is not generally a native POSIX NFS server in the same way an ONTAP SVM is.

If an NFS gateway, file gateway, or third-party product presents an NFS namespace backed by StorageGRID:

1. Troubleshoot the NFS export on the gateway first.
2. Check the gateway's export configuration, UID/GID mapping, ACL translation, and logs.
3. Verify gateway-to-StorageGRID connectivity and credentials.
4. Check StorageGRID Grid Manager health, alerts, node status, and service logs.
5. Test the object layer separately with an approved S3 client or API request.
6. Confirm whether the gateway provides POSIX consistency, locking, rename, and metadata semantics required by the application.

For ONTAP FabricPool or other S3 integration, separately verify the ONTAP object-store configuration, StorageGRID endpoint, certificates, credentials, DNS, routing, and time synchronization. FabricPool is not the same as mounting StorageGRID as an NFS filesystem.

---

## Linux filesystem and directory structure

Linux presents one directory tree rooted at `/`. Devices and remote filesystems are attached to this tree at mount points.

| Directory | Typical purpose |
|---|---|
| `/` | Filesystem root. Every path begins here. |
| `/bin` | Essential user commands; commonly merged into `/usr/bin` on modern distributions. |
| `/boot` | Bootloader files, kernel, and initramfs. |
| `/dev` | Device nodes such as disks, terminals, and pseudo-devices. |
| `/etc` | Host configuration, including NFS, identity, Kerberos, and security configuration. |
| `/home` | Normal users' home directories. |
| `/lib`, `/lib64` | Essential shared libraries and kernel modules; often linked into `/usr`. |
| `/media` | Automatic removable-media mount points. |
| `/mnt` | Temporary administrator mount point. Useful for NFS testing. |
| `/opt` | Optional third-party application software. |
| `/proc` | Virtual process and kernel information filesystem. |
| `/root` | Root user's home directory. |
| `/run` | Runtime state such as sockets and PID files; cleared at boot. |
| `/sbin` | Essential system administration commands; commonly merged into `/usr/sbin`. |
| `/srv` | Data served by system services. |
| `/sys` | Virtual kernel/device information filesystem. |
| `/tmp` | Temporary files; often has the sticky bit set. |
| `/usr` | Most userland programs, libraries, documentation, and shared read-only data. |
| `/var` | Variable data such as logs, spools, caches, and application state. |

Useful path commands:

```bash
pwd
cd /path/to/directory
ls -la
readlink -f /path/to/item
df -hT /path/to/item
du -xhd1 /path/to/directory
findmnt /path/to/item
```

On an NFS client, `df` and `findmnt` help confirm which remote filesystem actually contains a path. `df -T` shows the filesystem type, such as `nfs` or `nfs4`.

---

## Linux ownership and permissions

A directory listing such as:

```text
-rwxr-x--- 1 alice storage 4096 Sep 26 12:00 report.sh
```

means:

- `-`: regular file. A directory begins with `d`; a symbolic link begins with `l`.
- `rwx`: owner (`alice`) can read, write, and execute.
- `r-x`: group (`storage`) can read and execute, but not write.
- `---`: everyone else has no permissions.
- `1`: hard-link count.
- `alice`: owner name.
- `storage`: group name.
- `4096`: size in bytes for this example.
- The remaining fields show modification time and name.

### Permission bits

For regular files:

- `r` (4): read file contents.
- `w` (2): modify file contents.
- `x` (1): execute the file.

For directories:

- `r`: list directory names.
- `w`: create, delete, or rename entries, subject to other checks.
- `x`: traverse/search the directory and access known entries.

A directory generally needs `x` to be usable. A user may see a directory name but still be unable to access its contents if traversal permission is missing on any parent directory.

### Numeric modes

```bash
chmod 755 directory       # rwxr-xr-x
chmod 750 directory       # rwxr-x---
chmod 640 file            # rw-r-----
chmod 600 private-file    # rw-------
```

Symbolic examples:

```bash
chmod u+rwx,g+rx,o-rwx directory
chmod g+w shared-file
chown alice:storage file
chgrp storage directory
```

### Special bits

- **setuid (`4xxx`)**: executable runs with the file owner's effective UID.
- **setgid (`2xxx`)**: executable uses the file group's effective GID; on a directory, new files inherit the directory group.
- **sticky bit (`1xxx`)**: in a shared directory, users generally may delete only their own files. `/tmp` commonly uses mode `1777`.

Example shared directory:

```bash
mkdir /srv/team
chown root:storage /srv/team
chmod 2770 /srv/team
```

### ACLs and extended attributes

POSIX ACLs provide permissions for additional users and groups:

```bash
getfacl /path/to/file
setfacl -m u:alice:rwx /path/to/directory
setfacl -m g:storage:rwx /path/to/directory
```

Extended attributes can be inspected with:

```bash
getfattr -d -m- /path/to/file
```

NFS servers and gateways may translate ACLs and extended attributes differently. Check both client and server behavior when permissions appear inconsistent.

---

## Common problems and diagnostic paths

### Mount timeout or connection refused

1. Confirm DNS and routing.
2. Test TCP 2049.
3. For NFSv3, inspect `rpcinfo -p` and firewall rules.
4. Confirm ONTAP NFS is enabled on the SVM.
5. Confirm the export policy permits the client address and NFS version.
6. Check whether a firewall, security group, or network ACL blocks traffic.

### Mount succeeds but access is denied

1. Run `id` and compare numeric values with `ls -ln`.
2. Check every parent directory with `namei -l /mounted/path`.
3. Check export-policy read/write and `superuser` settings.
4. Check UNIX mode bits and NFSv4/POSIX ACLs.
5. Check root squashing and anonymous UID/GID mapping.
6. Check LDAP, AD, NIS, or local name-service consistency.
7. Check SELinux/AppArmor audit logs.

### NFSv4 user names appear as `nobody` or `user@domain`

Check `/etc/idmapd.conf`, the NFS identity-mapping domain, DNS, time synchronization, and name-service availability. Ensure the client and server use compatible identity sources and domains.

### `Stale file handle`

A file handle may become invalid after a server-side namespace change, volume restoration, export change, or snapshot/clone operation. Record the error and affected path, stop applications using it, then unmount and remount the filesystem if safe. Investigate ONTAP volume, snapshot, export-policy, and event history before treating repeated occurrences as a client-only issue.

### `NFS server not responding`

Check packet loss, MTU mismatches, interface or VLAN errors, firewall state, server load, and ONTAP event logs. Avoid immediately using force or lazy unmounts if applications may still be writing; collect logs first when possible.

### Locking problems

Check whether the issue is NFSv3 lock/statd related, NFSv4 state related, or an application-level lock problem. Review client kernel logs, `nfsstat`, ONTAP NFS connection/state information, and the application's locking behavior.

### Performance problems

Collect before changing options:

```bash
nfsstat -c
nfsstat -m
iostat -xz 1
vmstat 1
sar -n DEV 1
```

Compare protocol version, read/write sizes, latency, retransmits, server and client CPU, network errors, synchronous-write workload, locking contention, and ONTAP volume/aggregate performance. Avoid copying mount options from another environment without measuring the workload.

---

## Recommended evidence to collect

When escalating an NFS issue, include:

- Exact mount command and complete error text.
- Client OS and kernel version.
- NFS version and security flavor.
- `findmnt`, `nfsstat -m`, and relevant `dmesg`/`journalctl` output.
- `rpcinfo -p` and `showmount -e` results where applicable.
- Client IP, ONTAP SVM, volume, qtree, export policy, and policy rule output.
- Numeric ownership and mode/ACL output (`ls -ln`, `stat`, `getfacl`).
- Approximate start time, affected paths, and whether all clients or one client are affected.
- StorageGRID gateway and Grid Manager alerts/logs if a gateway or object backend is involved.

Redact credentials, Kerberos keys, tokens, and other sensitive information before sharing output.
