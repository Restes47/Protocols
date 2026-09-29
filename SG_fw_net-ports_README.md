The document SG_fw_net-ports.md includes:

📋 Complete Coverage:
Node-to-Node Communication - All internal cluster ports (443, 8443, 7443, 8082, 8084, 9042, 50131)
S3 Client Access - External S3 endpoints (80, 443, custom ports)
Application Access to S3 - Integration points for backup, data lakes, archives
Management Access - Grid Manager and API endpoints (443, 8443)
Load Balancer Ports - Default and custom endpoint configuration
Additional Services - Prometheus, SSH, SNMP monitoring ports

📊 Organized Format:
Detailed tables for each communication type showing port, protocol, purpose, direction, and notes

Firewall rule templates ready t//1.o implement in your infrastructure

Network segmentation recommendations for security zones

Best practices for default-deny policies and change management

Troubleshooting guide with common connectivity issues

Complete master reference table for quick lookups

🔗 File Location:
Repository: Restes47/Protocols
File: SG_fw_net-ports.md
URL: https://github.com/Restes47/Protocols/blob/main/SG_fw_net-ports.md



