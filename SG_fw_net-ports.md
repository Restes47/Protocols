# NetApp StorageGRID Firewall Rules & Network Ports Reference

**Document Version:** 1.0  
**Last Updated:** 2026-09-29  
**Purpose:** Comprehensive firewall configuration guide for NetApp StorageGRID deployments

---

## Table of Contents

1. [Overview](#overview)
2. [Node-to-Node Communication](#node-to-node-communication)
3. [S3 Client Access](#s3-client-access)
4. [Application Access to S3](#application-access-to-s3)
5. [Management Access to Grid](#management-access-to-grid)
6. [Load Balancer Ports](#load-balancer-ports)
7. [Additional Services](#additional-services)
8. [Complete Port Reference](#complete-port-reference)
9. [Firewall Configuration Best Practices](#firewall-configuration-best-practices)

---

## Overview

This document provides a complete reference for all network ports and firewall rules required for a fully functional NetApp StorageGRID deployment. Ports are categorized by communication type and purpose to simplify firewall configuration across different infrastructure components.

**Key Points:**
- All StorageGRID services rely on TCP/UDP protocols
- Default HTTPS port is 443; HTTP (port 80) is optional
- Custom load balancer endpoints can use any unassigned port
- Node-to-node communication must be bidirectional
- Management access should be restricted to authorized networks

---

## Node-to-Node Communication

Internal StorageGRID node communication enables cluster consensus, data replication, and inter-service messaging. These ports **must be open bidirectionally** between all node types (Admin, Storage, Gateway, Archive).

| Port  | Protocol | Service/Purpose                     | Nodes Involved          | Direction      | Notes                                    |
|-------|----------|-------------------------------------|-------------------------|-----------------|------------------------------------------|
| 443   | TCP      | Internal Services (HTTPS)           | All nodes               | Bidirectional   | Cluster management and inter-node API    |
| 8443  | TCP      | Admin Node Management               | Admin ↔ All nodes       | Bidirectional   | Admin-specific control plane             |
| 7443  | TCP      | Network Management System (NMS)     | Admin → Storage nodes   | Bidirectional   | Monitoring and telemetry                 |
| 8082  | TCP      | Object Communication & Replication  | Storage nodes           | Bidirectional   | Critical for data replication/erasure    |
| 8084  | TCP      | Cassandra Communication             | Storage nodes           | Bidirectional   | Database cluster (used by storage layer) |
| 9042  | TCP      | Cassandra (alternate)               | Storage nodes           | Bidirectional   | Internal database protocol               |
| 50131 | TCP      | Internode Communication             | All nodes               | Bidirectional   | RPC-style service communication          |

### Node-to-Node Firewall Rules

```
Source: All StorageGRID Nodes
Destination: All StorageGRID Nodes
Ports: 443, 7443, 8082, 8084, 9042, 50131
Protocol: TCP
Action: Allow
Direction: Bidirectional
```

---

## S3 Client Access

External S3 clients connect to StorageGRID through Gateway Nodes or via Load Balancer endpoints. These are **inbound** connections from client networks to the StorageGRID cluster.

| Port      | Protocol | Service             | Node Type       | Direction | Notes                                          |
|-----------|----------|---------------------|-----------------|-----------|------------------------------------------------|
| 80        | TCP      | S3 (HTTP)           | Gateway/LB      | Inbound   | Optional; often redirects to HTTPS             |
| 443       | TCP      | S3 (HTTPS)          | Gateway/LB      | Inbound   | Recommended; secure S3 API endpoint            |
| Custom*   | TCP      | S3 (Custom Port)    | LB Endpoint     | Inbound   | User-defined during load balancer setup        |

*Custom ports typically range from 1024-65535 and are defined in Grid Manager Load Balancer Endpoints.

### S3 Client Firewall Rules

```
Source: S3 Client Networks / Applications
Destination: StorageGRID Gateway/Load Balancer Nodes
Ports: 80 (optional), 443 (required)
Protocol: TCP
Action: Allow
Direction: Inbound
```

### Example Custom Load Balancer Ports
- **10080**: HTTP on custom port
- **10443**: HTTPS on custom port
- **9000**: S3 endpoint for legacy applications
- **8888**: S3 endpoint for development/testing

---

## Application Access to S3

Applications consuming StorageGRID S3 services require the same ports as S3 clients. This includes backup software, data lakes, archives, and application services.

| Port      | Protocol | Service                      | Source              | Notes                                       |
|-----------|----------|------------------------------|---------------------|---------------------------------------------|
| 443       | TCP      | S3 (HTTPS) - Primary         | Application servers | Recommended for production                  |
| 80        | TCP      | S3 (HTTP) - Legacy/Dev       | Application servers | Not recommended for production              |
| Custom LB | TCP      | S3 via Custom Endpoint       | Application servers | If using custom load balancer endpoint      |

### Application Firewall Rules

```
Source: Application Servers / Backup Infrastructure
Destination: StorageGRID S3 Endpoints (Gateway/LB)
Ports: 443 (and 80 if legacy support required)
Protocol: TCP
Action: Allow
Direction: Outbound (from app server perspective)
```

### Example Application Scenarios

| Application Type    | Ports Needed | Protocol | Notes                           |
|---------------------|--------------|----------|--------------------------------|
| Backup Software     | 443          | HTTPS    | Commvault, Veeam, NetBackup     |
| Data Lake / Spark   | 443          | HTTPS    | Hadoop, Delta Lake integrations |
| Archive Platform    | 443, custom  | HTTPS    | e.g., 10443 for dedicated pipe  |
| Development Tools   | 80, 443      | HTTP/S   | AWS CLI, Boto3, S3cmd           |
| CDN / Edge Cache    | 443          | HTTPS    | CloudFront, Akamai integration  |

---

## Management Access to Grid

Administrative users access StorageGRID Grid Manager (web UI) and REST API through secure endpoints on Admin Nodes.

| Port  | Protocol | Service                    | Access Type | Direction | Notes                                        |
|-------|----------|----------------------------|-------------|-----------|----------------------------------------------|
| 443   | TCP      | Grid Manager (HTTPS)       | Admin UI    | Inbound   | Primary management interface                 |
| 8443  | TCP      | Untrusted Network Access   | Admin UI    | Inbound   | For admin access from untrusted networks     |
| 443   | TCP      | Grid Management REST API   | API calls   | Inbound   | Used by automation and scripts               |

### Management Firewall Rules

```
Source: Administrative Networks (should be restricted)
Destination: StorageGRID Admin Node
Ports: 443 (trusted), 8443 (untrusted networks)
Protocol: TCP
Action: Allow
Direction: Inbound
```

### Trusted vs. Untrusted Network Access

| Network Type | Port  | Recommended Use                                   |
|--------------|-------|---------------------------------------------------|
| Trusted      | 443   | Internal IT infrastructure, secured networks      |
| Untrusted    | 8443  | External partners, cloud-based access, VPNs       |

---

## Load Balancer Ports

StorageGRID Load Balancer Endpoints provide flexible, multi-protocol access to S3/Swift services. Each endpoint can be independently configured.

### Default Load Balancer Endpoints

| Port  | Protocol | Service          | Node Type       | Direction | Notes                                     |
|-------|----------|------------------|-----------------|-----------|-------------------------------------------|
| 80    | TCP      | S3 HTTP (default)| LB Endpoint     | Inbound   | Often redirected to HTTPS                 |
| 443   | TCP      | S3 HTTPS (default)| LB Endpoint     | Inbound   | Secure S3 endpoint                        |

### Custom Load Balancer Endpoints

Custom endpoints can be created for:
- High-capacity S3 workloads on separate ports
- S3 access from specific application tiers
- Multi-tenancy isolation
- Legacy application compatibility

| Port        | Protocol | Use Case                                |
|-------------|----------|------------------------------------------|
| 9000-9100   | TCP      | Development/Testing endpoints           |
| 10000-10999 | TCP      | Production custom endpoints             |
| 20000-29999 | TCP      | Multi-tenant endpoints                  |
| Custom      | TCP      | Any unassigned high-numbered port       |

### Load Balancer Firewall Rules Template

```
Source: Application/Client Networks
Destination: StorageGRID Load Balancer Nodes
Ports: 443, 80, + Custom LB Endpoint Ports
Protocol: TCP
Action: Allow
Direction: Inbound
Example: Allow 10.0.0.0/8 → 192.168.1.0/24 on ports 80, 443, 10443
```

---

## Additional Services

### Prometheus Metrics & Monitoring

| Port  | Protocol | Service                 | Access From     | Direction | Notes                          |
|-------|----------|-------------------------|-----------------|-----------|--------------------------------|
| 9443  | TCP      | Prometheus Metrics      | Monitoring nets | Inbound   | For Prometheus scraping        |

### SSH (Rare - Emergency Only)

| Port  | Protocol | Service                 | Access From     | Direction | Notes                          |
|-------|----------|-------------------------|-----------------|-----------|--------------------------------|
| 22    | TCP      | SSH (Emergency console) | Admin networks  | Inbound   | **Not recommended** for normal operations |

### SNMP (Optional)

| Port  | Protocol | Service                 | Access From     | Direction | Notes                          |
|-------|----------|-------------------------|-----------------|-----------|--------------------------------|
| 161   | UDP      | SNMP (if enabled)       | Monitoring nets | Inbound   | Optional network monitoring    |

---

## Complete Port Reference

### Comprehensive Master List

| Port   | Protocol | Service                          | Communication Type      | Direction      | Node Type(s)     |
|--------|----------|----------------------------------|-------------------------|-----------------|------------------|
| 22     | TCP      | SSH (Emergency)                  | Management              | Inbound         | All              |
| 80     | TCP      | HTTP (S3, optional)              | Client/App Access       | Inbound         | Gateway/LB       |
| 161    | UDP      | SNMP (optional)                  | Monitoring              | Inbound         | All              |
| 443    | TCP      | HTTPS (S3, API, Mgmt, Internode)| Multi-purpose           | Bidirectional   | All              |
| 7443   | TCP      | NMS                              | Node-to-Node            | Bidirectional   | Storage/Admin    |
| 8082   | TCP      | Object Communication             | Node-to-Node            | Bidirectional   | Storage          |
| 8084   | TCP      | Cassandra                        | Node-to-Node            | Bidirectional   | Storage          |
| 8443   | TCP      | Admin Node Management            | Management/Internode    | Bidirectional   | Admin/Gateway    |
| 9042   | TCP      | Cassandra (alternate)            | Node-to-Node            | Bidirectional   | Storage          |
| 9443   | TCP      | Prometheus                       | Monitoring              | Inbound         | All              |
| 50131  | TCP      | Internode Communication          | Node-to-Node            | Bidirectional   | All              |
| Custom | TCP      | Load Balancer Endpoints          | Client/App Access       | Inbound         | Gateway/LB       |

---

## Firewall Configuration Best Practices

### 1. **Default Deny Policy**
```
Default: Deny all inbound traffic
Allow: Only explicitly required ports and source networks
Document: All firewall rules with business justification
```

### 2. **Network Segmentation**

Separate StorageGRID traffic by security zone:

```
Management Network:  10.0.1.0/24     (restricted access to Grid Manager)
Internode Network:   10.0.2.0/24     (only internal SGW communication)
S3 Client Network:   10.0.3.0/24     (external/application S3 access)
Monitoring Network:  10.0.4.0/24     (Prometheus, SNMP, alerts)
```

### 3. **Source/Destination Rules**

**Example: Production S3 Client Access**
```
Name: Allow-S3-Clients
Source: 10.0.3.0/24 (S3 client network)
Destination: 192.168.1.10, 192.168.1.11 (Gateway nodes)
Ports: 80, 443
Protocol: TCP
Action: Allow
Log: Yes
```

**Example: Node-to-Node Communication**
```
Name: Allow-Internode-Comm
Source: 192.168.1.0/24 (all SGW nodes)
Destination: 192.168.1.0/24 (all SGW nodes)
Ports: 443, 7443, 8082, 8084, 9042, 50131
Protocol: TCP
Action: Allow
Log: Yes
```

### 4. **High Availability Considerations**

If using multiple Admin Nodes or gateway failover:
- Open all internode ports between all Admin and Gateway instances
- Ensure Load Balancer nodes can reach all Gateway Nodes on S3 ports
- Test failover with firewall rules in place

### 5. **Monitoring and Logging**

- Enable firewall logging for all StorageGRID traffic
- Monitor blocked connection attempts
- Review logs monthly for unexpected traffic patterns
- Set alerts for repeated connection failures to management ports

### 6. **Change Management**

When adding custom load balancer endpoints:
1. Document the port and purpose in change request
2. Add firewall rule(s) for the new port
3. Test connectivity from client network
4. Update this reference document
5. Communicate changes to network and security teams

### 7. **Security Hardening**

- **Disable HTTP (port 80)** if not required; redirect to HTTPS
- **Restrict management ports (443/8443)** to authorized subnets only
- **Use firewalls at multiple levels** (network, host, container)
- **Review credentials** when opening new ports
- **Implement rate limiting** on management endpoints

### 8. **Disaster Recovery**

- Keep this firewall rule document in version control
- Test firewall rules as part of DR procedures
- Verify intra-site and inter-site node communication during testing
- Document any environment-specific port variations

---

## Troubleshooting Connectivity Issues

### Checklist for Firewall Verification

- [ ] Verify bidirectional connectivity for node-to-node ports
- [ ] Confirm inbound access from client networks to S3 endpoints
- [ ] Check administrative network can reach port 443 (management)
- [ ] Validate custom load balancer endpoint ports are open
- [ ] Ensure DNS resolution works for StorageGRID hostnames
- [ ] Test with `telnet` or `nc` to specific ports and nodes
- [ ] Review firewall logs for denied connections
- [ ] Verify SSL/TLS certificate validity on management endpoints

### Common Issues and Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| **Missing internode ports** | Nodes unable to join cluster | Open all internode ports (443, 8082, 8084, etc.) bidirectionally |
| **S3 client timeout** | Application cannot reach S3 | Verify S3 endpoint ports (80/443) open from client network |
| **Management UI unreachable** | Cannot access Grid Manager | Check port 443 or 8443 open from admin network |
| **Data replication failure** | Replication tasks stall | Ensure port 8082 open between storage nodes |
| **Monitoring gaps** | Missing Prometheus metrics | Verify port 9443 open to monitoring infrastructure |

---

## Related Documentation

- [NetApp StorageGRID Network Ports Reference](https://docs.netapp.com/en-us/storagegrid-116/network/storagegrid-network-ports.html)
- [StorageGRID Firewall Considerations](https://docs.netapp.com/en-us/storagegrid-116/admin/firewall-considerations.html)
- [Configuring Load Balancer Endpoints](https://docs.netapp.com/en-us/storagegrid-116/admin/configuring-load-balancer-endpoints.html)
- [Admin Node Virtual IP Addressing](https://docs.netapp.com/en-us/storagegrid-116/admin/admin-node-virtual-ip.html)

---

## Revision History

| Version | Date       | Author    | Changes                              |
|---------|------------|-----------|--------------------------------------|
| 1.0     | 2026-09-29 | Reference | Initial comprehensive port reference |

---

**Document Status:** Active Reference Guide  
**Classification:** Internal Technical Documentation  
**Last Reviewed:** 2026-09-29

---

*For questions or updates to this document, please submit a pull request or issue with the specific port or service requiring clarification.*
