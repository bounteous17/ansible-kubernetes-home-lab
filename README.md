# ansible-kubernetes-home-lab

Ansible-based k3s cluster deployment for home lab infrastructure.

## Prerequisites

- Ansible 2.10+
- SSH access to target hosts
- Target hosts running a supported Linux distribution

## Installation

### 1. Install the k3s-ansible collection

```bash
ansible-galaxy collection install git+https://github.com/k3s-io/k3s-ansible.git
```

### 2. Create inventory.yml

Create an `inventory.yml` file (not tracked in git for security):

```yaml
k3s_cluster:
  children:
    server:
      hosts:
        <server-ip-1>:
        <server-ip-2>:
    agent:
      hosts:
        <agent-ip-1>:
        <agent-ip-2>:

  vars:
    ansible_port: 22
    ansible_user: <your-ssh-user>
    ansible_become_password: "<your-sudo-password>"
    k3s_version: v1.35.0+k3s1
    token: "<generate-with-openssl-rand-base64-64>"
    api_endpoint: "{{ hostvars[groups['server'][0]]['ansible_host'] | default(groups['server'][0]) }}"
```

### 3. Deploy the cluster

```bash
ansible-playbook k3s.orchestration.site -i inventory.yml
```

## Cluster Operations

### Install open-iscsi (Required for Longhorn)

Before deploying Longhorn storage, install `open-iscsi` on all cluster nodes:

```bash
ansible-playbook install-open-iscsi.yml -i inventory.yml
```

**What it does:**

1. Detects the OS on each node and installs `open-iscsi` using the appropriate package manager:
   - **Arch Linux** (server nodes): Uses `pacman`
   - **Raspbian** (agent nodes): Uses `apt`
2. Enables and starts the `iscsid` service on all nodes
3. Verifies the service is running

**Note:** This is required for Longhorn distributed storage to function properly. The playbook automatically handles the mixed OS environment (Arch Linux servers and Raspbian agents).

### Install nfs-utils (Required for Longhorn)

Longhorn requires NFS utilities for ReadWriteMany (RWX) volume support. Install on all cluster nodes:

```bash
ansible-playbook install-nfs-utils.yml -i inventory.yml
```

**What it does:**

1. Detects the OS on each node and installs NFS utilities using the appropriate package manager:
   - **Arch Linux** (server nodes): Installs `nfs-utils` via `pacman`
   - **Raspbian** (agent nodes): Installs `nfs-common` via `apt`
2. Enables and starts the `rpcbind` service on all nodes
3. Verifies the installation

**Note:** This is required for Longhorn to create disks on nodes. Without this, you'll see "Missing packages: [nfs-utils]" in the Longhorn UI.

### Label Storage Nodes (Required for Longhorn)

Label server nodes (master nodes with SSDs) so Longhorn only uses them for storage:

```bash
ansible-playbook label-storage-nodes.yml -i inventory.yml
```

**What it does:**

1. Labels all server nodes with `storage-node=true`
2. This label is used by Longhorn to restrict storage to nodes with SSDs
3. Agent nodes (Raspbian) are excluded from storage since they don't have sufficient speed

**Note:** This must be run before deploying Longhorn. The Longhorn Helm chart is configured to only use nodes with the `storage-node=true` label.

### Load dm_crypt Kernel Modules (Optional - for Longhorn Encryption)

If you plan to use encrypted volumes in Longhorn, load the required kernel modules:

```bash
ansible-playbook load-dm-crypt-modules.yml -i inventory.yml
```

**What it does:**

1. Loads `dm_mod` and `dm_crypt` kernel modules on all nodes
2. Configures modules to load automatically on boot:
   - **Arch Linux**: Adds to `/etc/modules-load.d/dm-modules.conf`
   - **Raspbian**: Adds to `/etc/modules`
3. Verifies modules are loaded

**Note:** This is optional. If you don't need encrypted volumes, you can ignore the warning in Longhorn UI. The playbook handles the mixed OS environment automatically.

### Safe Cluster Reboot

The `reboot-cluster.yml` playbook safely reboots all cluster nodes in the correct order:

```bash
ansible-playbook reboot-cluster.yml -i inventory.yml
```

**What it does:**

1. **Gathers node information** - Caches hostnames for kubectl operations
2. **Drains and reboots agent nodes** - One at a time, evicting workloads gracefully, then reboots and uncordons
3. **Drains and reboots secondary servers** - One at a time, maintaining quorum as long as possible, then reboots and uncordons
4. **Drains and reboots primary server last** - Cordons, drains, reboots, and uncordons the primary server
5. **Verifies cluster health** - Checks that all nodes are back online and Ready

**Features:**

- Automatically drains nodes before reboot (workloads are rescheduled)
- Waits for nodes to come back online after reboot
- Automatically uncordons nodes so they can accept new workloads
- Verifies k3s services are running before uncordoning
- Checks cluster health at the end

**Options:**

- Add `-v` for verbose output including drain/uncordon status details
- The playbook uses `serial: 1` to process nodes sequentially, ensuring safe reboot order
- Reboot timeout is 600 seconds (10 minutes) per node

**Notes:**

- Workloads are given a 60-second grace period to terminate before drain completes
- Drain operations timeout after 300 seconds
- The cluster will be temporarily unavailable when the primary server reboots
- Requires `become: true` (sudo) for reboot commands

### Graceful Cluster Shutdown

The `poweroff.yml` playbook safely shuts down the entire k3s cluster in the correct order:

```bash
ansible-playbook poweroff.yml -i inventory.yml
```

**What it does:**

1. **Gathers node information** - Caches hostnames for kubectl operations
2. **Drains and shuts down agent nodes** - One at a time, evicting workloads gracefully
3. **Drains and shuts down secondary servers** - One at a time, maintaining quorum as long as possible
4. **Shuts down primary server last** - Cordons the node and performs final shutdown

**Options:**

- Add `-v` for verbose output including drain status details
- The playbook uses `serial: 1` to process nodes sequentially, ensuring safe shutdown order

**Notes:**

- Workloads are given a 60-second grace period to terminate
- Drain operations timeout after 300 seconds
- DaemonSets and emptyDir data are handled automatically
- Requires `become: true` (sudo) for shutdown commands

## Cluster Architecture

- **Server nodes**: Control plane nodes running k3s server
- **Agent nodes**: Worker nodes running k3s agent
- **External database**: PostgreSQL backend for HA configuration (optional)
