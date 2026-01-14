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