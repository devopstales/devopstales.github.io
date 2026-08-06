---
title: Ceph Debugging on Proxmox: Essential Commands and Troubleshooting
url: https://devopstales.github.io/virtualization/proxmox-ceph-debugging/
date: 2026-03-04
keywords: proxmox ceph, ceph debugging, ceph commands
---


When running Ceph on Proxmox, issues can arise at any layer of the storage stack. Knowing the right commands to diagnose and resolve problems is essential for maintaining a healthy cluster. This guide covers the most useful Ceph commands for debugging and troubleshooting on Proxmox.

<!--more-->

## Cluster Health and Status

The first step in any troubleshooting session is to check the overall cluster health.

```bash
ceph -s
```

This shows a summary including health status, OSD/MON/MGR counts, I/O rates, and placement group states. For more detailed health information:

```bash
ceph health detail
```

To check disk usage across pools:

```bash
ceph df
```

## OSD Diagnostics

OSDs (Object Storage Daemons) are often the source of issues. Start by viewing the hierarchical layout:

```bash
ceph osd tree
```

Check the up/in status of all OSDs:

```bash
ceph osd status
```

For detailed usage information including space and performance weight:

```bash
ceph osd df
```

List all storage pools:

```bash
ceph osd pool ls
```

Get per-pool statistics for performance monitoring:

```bash
ceph osd pool stats
```

### OSD Maintenance Operations

When performing maintenance on an OSD, mark it out first:

```bash
ceph osd out osd.3
```

After maintenance, bring it back into the cluster:

```bash
ceph osd in osd.3
```

To manually adjust an OSD's relative data weight:

```bash
ceph osd crush reweight osd.2 0.8
```

## Monitor and Manager Status

For troubleshooting monitor quorum issues:

```bash
ceph quorum_status
```

Check the manager daemon status and enabled modules:

```bash
ceph mgr dump
```

## Placement Groups (PGs)

Placement groups are fundamental to Ceph's data distribution. Check their state:

```bash
ceph pg stat
```

For detailed information about all placement groups:

```bash
ceph pg dump
```

List PGs belonging to a specific pool:

```bash
ceph pg ls-by-pool <POOL-NAME>
```

## RBD Pool and Image Management

List RBD images in a pool:

```bash
rbd ls -p vm-storage
```

Show details of a specific block image:

```bash
rbd info -p vm-storage vm-100-disk-0
```

List all snapshots for a given image:

```bash
rbd snap ls vm-100-disk-0
```

Resize an image (expand only):

```bash
rbd resize vm-100-disk-0 --size 20480
```

## Performance Testing and Maintenance

Benchmark all OSDs to identify performance bottlenecks:

```bash
ceph tell osd.* bench
```

Manually trigger a scrub on an OSD for data consistency checks:

```bash
ceph osd scrub osd.0
```

Check if automatic balancing is active:

```bash
ceph balancer status
```

Enable optional manager modules:

```bash
ceph mgr module enable dashboard
```

## Authentication and Keyring Management

List all Ceph authentication keys:

```bash
ceph auth list
```

Get a specific client's authentication key:

```bash
ceph auth get client.admin
```

Remove a client key:

```bash
ceph auth del client.radosgw.pve1
```

## Cleanup and Decommissioning

Completely remove an OSD from the cluster:

```bash
ceph osd purge osd.3 --yes-i-really-mean-it
```

Remove a monitor that has been physically removed or decommissioned:

```bash
ceph mon remove mon.pve3
```

## Proxmox-Specific Commands

Proxmox provides wrapper commands that integrate with its configuration system:

```bash
pveceph status
pveceph pool ls
pveceph osd create /dev/sdb
pveceph install --version quincy
```

These commands help maintain UI synchronization and work with Proxmox's configuration files.

## Ceph Dashboard Installation

The Ceph dashboard provides a web-based interface for monitoring and managing your cluster.

### Install Dashboard Package

Run on **all manager nodes**:

```bash
apt install ceph-mgr-dashboard -y
```

### Enable Dashboard Module

Run on **any manager node**:

```bash
ceph mgr module enable dashboard
ceph mgr module ls | grep dashboard  # Verify
```

### SSL Configuration

For a quick homelab setup, disable SSL:

```bash
ceph config set mgr mgr/dashboard/ssl false
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

For production, use manual SSL setup:

```bash
# Generate self-signed certificate
openssl req -newkey rsa:2048 -nodes -x509 \
  -keyout /root/dashboard-key.pem \
  -out /root/dashboard-crt.pem \
  -sha512 -days 3650 \
  -subj "/CN=IT/O=ceph-mgr-dashboard" -utf8

# Install certificates
ceph config-key set mgr/dashboard/key -i /root/dashboard-key.pem
ceph config-key set mgr/dashboard/crt -i /root/dashboard-crt.pem

# Enable SSL and restart
ceph config set mgr mgr/dashboard/ssl true
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

### Create Admin User

For homelab environments, disable password policies:

```bash
ceph dashboard set-pwd-policy-check-complexity-enabled false
ceph dashboard set-pwd-policy-enabled false
```

Create the admin user:

```bash
echo "admin" > ./password
ceph dashboard ac-user-create admin -i ./password administrator --force
```

### Access the Dashboard

Get the dashboard URL:

```bash
ceph mgr services
```

| Protocol | Port |
|----------|------|
| HTTP     | 8080 |
| HTTPS    | 8443 |

Access via `https://<proxmox-node>:8443/#/dashboard` with username `admin` and your configured password.

> **Note:** The dashboard runs on the node with the active `ceph-mgr`. The URL may change on failover.

## Quick Troubleshooting Workflow

When issues arise, follow this systematic approach:

1. **Check cluster health** - Start with `ceph -s` or `ceph health detail`
2. **Identify problematic OSDs** - Use `ceph osd tree` and `ceph osd status`
3. **Check disk usage** - Run `ceph df` and `ceph osd df`
4. **Review PG states** - Execute `ceph pg stat` to find stuck placement groups
5. **Perform maintenance** - Use `ceph osd out` before work, then `ceph osd in` after
6. **Benchmark performance** - Run `ceph tell osd.* bench` to identify slow OSDs

## Common Issues and Solutions

### OSD Marked Out Unexpectedly

If an OSD is marked out without intervention, check the logs:

```bash
journalctl -u ceph-osd@osd.0 -f
```

### Placement Groups Stuck

PGs stuck in `degraded` or `undersized` state often indicate OSD failures. Use `ceph pg dump` to identify affected groups and their OSDs.

### Monitor Quorum Loss

If monitors lose quorum, check network connectivity between nodes and verify monitor health with `ceph quorum_status`.

### Slow Performance

Use `ceph osd df` to identify imbalanced OSDs and `ceph tell osd.* bench` to benchmark individual daemons. Consider enabling the balancer module for automatic rebalancing.

## Conclusion

Mastering these Ceph commands will significantly reduce troubleshooting time on your Proxmox cluster. The key is to start with high-level health checks and progressively drill down to specific components. Regular monitoring and understanding normal cluster behavior will help you quickly identify when something deviates from the expected state.

For ongoing monitoring, consider enabling the Ceph dashboard for a visual overview of cluster health, performance metrics, and management capabilities. However, command-line tools remain essential for deep debugging and automation scenarios.

