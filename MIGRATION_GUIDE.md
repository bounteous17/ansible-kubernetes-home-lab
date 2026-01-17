# Cluster Node Role Migration Guide

## Current Configuration
- **Server nodes**: 192.168.11.50, 192.168.11.100
- **Agent nodes**: 192.168.11.51, 192.168.11.101

## Target Configuration
- **Server nodes**: 192.168.11.50, 192.168.11.51
- **Agent nodes**: 192.168.11.100, 192.168.11.101

## Migration Steps

### Prerequisites
- External PostgreSQL database (already configured) - this makes migration safer
- Backup any important data/workloads
- Ensure you have kubectl access to the cluster

### Step 1: Verify Current Cluster State

```bash
# Check current node roles
kubectl get nodes -o wide

# Check for any workloads on nodes being migrated
kubectl get pods --all-namespaces -o wide | grep -E "192.168.11.51|192.168.11.100"
```

### Step 2: Convert 192.168.11.100 (Server → Agent)

**2.1. Drain the node**
```bash
# Get the node hostname first
kubectl get nodes

# Drain workloads from 192.168.11.100
kubectl drain <node-hostname-100> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=60
```

**2.2. Remove node from cluster**
```bash
# On 192.168.11.100, stop and uninstall k3s
ssh 192.168.11.100
sudo systemctl stop k3s
sudo /usr/local/bin/k3s-uninstall.sh
```

**2.3. Update inventory.yml** (temporarily, before running playbook)
Move 192.168.11.100 from `server` to `agent` section.

**2.4. Reinstall as agent**
```bash
# From ansible-kubernetes-home-lab directory
ansible-playbook k3s.orchestration.site -i inventory.yml --limit agent
```

**2.5. Verify**
```bash
kubectl get nodes
# Should show 192.168.11.100 as agent (no control-plane role)
```

### Step 3: Convert 192.168.11.51 (Agent → Server)

**3.1. Drain the node**
```bash
# Get the node hostname first
kubectl get nodes

# Drain workloads from 192.168.11.51
kubectl drain <node-hostname-51> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=60
```

**3.2. Remove node from cluster**
```bash
# On 192.168.11.51, stop and uninstall k3s
ssh 192.168.11.51
sudo systemctl stop k3s-agent
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

**3.3. Update inventory.yml**
Move 192.168.11.51 from `agent` to `server` section.

**3.4. Reinstall as server**
```bash
# From ansible-kubernetes-home-lab directory
ansible-playbook k3s.orchestration.site -i inventory.yml --limit server
```

**3.5. Verify**
```bash
kubectl get nodes
# Should show 192.168.11.51 as server (with control-plane role)
```

### Step 4: Final Verification

```bash
# Verify all nodes are in correct roles
kubectl get nodes -o wide

# Verify cluster is healthy
kubectl get pods --all-namespaces

# Re-label storage nodes (if Longhorn is installed)
ansible-playbook label-storage-nodes.yml -i inventory.yml
```

## Important Notes

1. **Workloads will be rescheduled**: When draining nodes, workloads will automatically move to other nodes. Ensure you have sufficient capacity.

2. **Longhorn volumes**: If Longhorn is installed, ensure volumes are not affected. The external database helps maintain cluster state.

3. **Order matters**: Convert one node at a time to maintain quorum and availability.

4. **Backup**: Consider backing up important data before migration.

5. **Timing**: Perform during maintenance window if possible.

## Rollback Plan

If something goes wrong:

1. Stop the migration process
2. Restore the original inventory.yml
3. Reinstall the affected node with its original role
4. Verify cluster health
