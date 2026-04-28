# redis-ansible

Production-ready Redis Sentinel cluster with dual HAProxy + Keepalived (VRRP) for high availability, automated with Ansible.

---

## Architecture

```
                          Clients
                             │
                             ▼
                    ┌────────────────┐
                    │  Virtual IP    │
                    │ 192.168.100.200│  ◄── Keepalived VRRP floats this IP
                    └───────┬────────┘         between haproxy-1 and haproxy-2
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
   ┌─────────────────────┐   ┌─────────────────────┐
   │      haproxy-1      │   │      haproxy-2      │
   │   192.168.122.14    │   │   192.168.122.15    │
   │  Keepalived MASTER  │   │  Keepalived BACKUP  │
   │  HAProxy :6379      │   │  HAProxy :6379      │
   │  Stats UI  :8585    │   │  Stats UI  :8585    │
   └─────────┬───────────┘   └───────────┬─────────┘
             │                           │
             └─────────────┬─────────────┘
                           │
          HAProxy health-checks all 3 Redis nodes.
          Only the current master passes (role:master check).
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐
│  redis-server-1  │ │redis-server-2│ │redis-server-3│
│  192.168.100.11  │ │192.168.100.12│ │192.168.100.13│
│  role: MASTER    │ │role: SLAVE   │ │role: SLAVE   │
│  Redis   :6379   │ │Redis  :6379  │ │Redis  :6379  │
│  Sentinel:26379  │ │Sentinel:26379│ │Sentinel:26379│
└──────────────────┘ └──────────────┘ └──────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
            Redis Sentinel quorum = 2
            If master is unreachable for 5s,
            2 sentinels vote → elect new master
            → slaves repoint → HAProxy health
              check switches traffic automatically
```

### Failover flow

```
Master down
    │
    ├─► Sentinel detects (down-after-milliseconds: 5000)
    │
    ├─► 2 of 3 sentinels agree → failover starts
    │
    ├─► One slave promoted to master
    │
    ├─► Remaining slave repoints to new master
    │
    └─► HAProxy health check (role:master) fails on old master,
        passes on new master → traffic switches automatically
        within 1–2 seconds (check inter 1s)

HAProxy node down
    │
    ├─► Keepalived detects HAProxy process gone (chk_haproxy, interval: 2s)
    │
    └─► VIP moves to backup HAProxy node within ~4 seconds
```

---

## Prerequisites

- Ansible >= 2.9 on your control machine
- 5 Ubuntu servers reachable over SSH
- The SSH user must have passwordless sudo, or pass `-K` at runtime
- All nodes must be able to reach each other on the private network

---

## Configuration

### 1. Set node IPs

Edit `inventory/hosts.yml` and fill in real IPs for all 5 nodes:

```yaml
all:
  vars:
    ansible_user: <ssh-user>
    ansible_port: 22
    ansible_python_interpreter: /usr/bin/python3
    domain: cluster.local

  children:
    redis:
      hosts:
        redis-server-1:
          ansible_host: <PUBLIC_IP>
          private_ip: <PRIVATE_IP>
          role: master
        redis-server-2:
          ansible_host: <PUBLIC_IP>
          private_ip: <PRIVATE_IP>
          role: slave
        redis-server-3:
          ansible_host: <PUBLIC_IP>
          private_ip: <PRIVATE_IP>
          role: slave

    haproxy:
      hosts:
        haproxy-1:
          ansible_host: <PUBLIC_IP>
          private_ip: <PRIVATE_IP>
          keepalived_state: MASTER
          keepalived_priority: 101
        haproxy-2:
          ansible_host: <PUBLIC_IP>
          private_ip: <PRIVATE_IP>
          keepalived_state: BACKUP
          keepalived_priority: 100
```

### 2. Set non-secret variables

Edit `inventory/group_vars/all/vars.yml`:

| Variable | Description |
|---|---|
| `haproxy_vip` | Floating virtual IP — must be a free IP in your private subnet |
| `haproxy_vip_interface` | Network interface on HAProxy nodes — check with `ip link show` |
| `keepalived_vrid` | VRRP router ID (1–255) — must be unique on your LAN segment |
| `haproxy_stats_user` | HAProxy stats UI login username |
| `use_iran` | `"true"` to apply Iran-specific DNS workaround, otherwise `"false"` |

### 3. Set secrets with Ansible Vault

Secrets live in `inventory/group_vars/all/vault.yml` and are referenced in `vars.yml` as `{{ vault_* }}`. This file must be encrypted before committing.

**First-time setup:**

```bash
# 1. Create your vault password file from the example (it is already in .gitignore)
cp .vault_pass.example .vault_pass
chmod 600 .vault_pass
```

Open `.vault_pass` and replace the placeholder with your chosen master password — this is the single password that unlocks all secrets. `ansible.cfg` already points to it via `vault_password_file = .vault_pass`, so every `ansible-playbook` and `ansible-vault` command picks it up automatically.

```bash
# 2. Encrypt the vault file using that password
ansible-vault encrypt inventory/group_vars/all/vault.yml

# 3. Edit the encrypted vault file and set your real secrets
ansible-vault edit inventory/group_vars/all/vault.yml
```

The vault file contains three variables — set real values for all of them:

```yaml
vault_redis_password: "your-strong-redis-password"
vault_keepalived_auth_pass: "your-keepalived-password"
vault_haproxy_stats_password: "your-stats-ui-password"
```

**Day-to-day vault commands:**

```bash
# View the decrypted contents
ansible-vault view inventory/group_vars/all/vault.yml

# Edit a secret (opens your $EDITOR, saves re-encrypted)
ansible-vault edit inventory/group_vars/all/vault.yml

# Change the vault master password
ansible-vault rekey inventory/group_vars/all/vault.yml

# Decrypt to plaintext (only when needed, re-encrypt after)
ansible-vault decrypt inventory/group_vars/all/vault.yml
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

**If you have not set up `.vault_pass`**, pass the password interactively instead:

```bash
ansible-playbook redis.yaml --ask-vault-pass
```

---

## Installation

### 1. Verify SSH connectivity

```bash
ansible -i inventory/hosts.yml all -m ping
```

All 5 nodes should return `pong`. (`ansible.cfg` sets the inventory automatically.)

### 2. Run the full playbook

```bash
ansible-playbook redis.yaml
```

Or run one layer at a time for a safer first install:

```bash
# System prep on all 5 nodes
ansible-playbook redis.yaml --tags preinstall

# Redis + Sentinel on the 3 Redis nodes
ansible-playbook redis.yaml --tags redis

# HAProxy + Keepalived on the 2 HAProxy nodes
ansible-playbook redis.yaml --tags haproxy
```

If your user requires a sudo password, add `-K` to any command above:

```bash
ansible-playbook redis.yaml -K
```

### Re-running after a Redis failover

Sentinel rewrites `sentinel.conf` on each node when it promotes a new master. Re-running the playbook will **not** overwrite that file by default — the cluster stays in its current state. To force a full reset of the sentinel config back to the original master, pass:

```bash
ansible-playbook redis.yaml --tags redis -e force_sentinel_config=true
```

---

## Verification

### Check Redis replication

```bash
# Should return role:master
redis-cli -h 192.168.100.11 -a <password> info replication | grep -E 'role|connected_slaves'

# Should return role:slave
redis-cli -h 192.168.100.12 -a <password> info replication | grep role
redis-cli -h 192.168.100.13 -a <password> info replication | grep role
```

### Check Sentinel state

```bash
# Run on any Redis node — should show mymaster and its slaves
redis-cli -h 192.168.100.11 -p 26379 sentinel master mymaster
redis-cli -h 192.168.100.11 -p 26379 sentinel slaves mymaster
redis-cli -h 192.168.100.11 -p 26379 sentinel sentinels mymaster
```

### Check the VIP is active

```bash
ping 192.168.100.200

# Should succeed — write through VIP, read back
redis-cli -h 192.168.100.200 -a <password> set test "hello"
redis-cli -h 192.168.100.200 -a <password> get test
```

### HAProxy stats UI

Open in a browser: `http://192.168.100.200:8585`

Login with `admin` / your stats password. The `back_redis` backend shows all 3 Redis nodes. Only the current master is green — slaves are red because the health check verifies `role:master`. This is expected.

---

## Testing Failover

### Test 1 — Redis master failure

```bash
# 1. Write a key through the VIP
redis-cli -h 192.168.100.200 -a <password> set failover-test "before"

# 2. Stop Redis on the master
ssh milad@192.168.100.11 "sudo systemctl stop redis-server"

# 3. Watch Sentinel elect a new master (takes ~5-10 seconds)
watch -n1 'redis-cli -h 192.168.100.12 -p 26379 sentinel master mymaster | grep -E "name|ip|port|flags"'

# 4. Confirm traffic routes to the new master — key must still be readable
redis-cli -h 192.168.100.200 -a <password> get failover-test

# 5. Confirm which node is now master
redis-cli -h 192.168.100.12 -a <password> info replication | grep role
redis-cli -h 192.168.100.13 -a <password> info replication | grep role

# 6. Restore the old master (it will rejoin as a slave)
ssh milad@192.168.100.11 "sudo systemctl start redis-server"
redis-cli -h 192.168.100.11 -a <password> info replication | grep role
```

### Test 2 — HAProxy node failure (VIP failover)

```bash
# 1. Confirm VIP is on haproxy-1
ssh milad@192.168.122.14 "ip addr show | grep 192.168.100.200"

# 2. Stop HAProxy on haproxy-1
ssh milad@192.168.122.14 "sudo systemctl stop haproxy"

# 3. Keepalived detects the failure within ~4 seconds and moves the VIP
#    Watch the VIP appear on haproxy-2
watch -n1 'ssh milad@192.168.122.15 "ip addr show | grep 192.168.100.200"'

# 4. Traffic through VIP still works
redis-cli -h 192.168.100.200 -a <password> ping

# 5. Restore haproxy-1 — VIP stays on haproxy-2 until haproxy-2 fails
#    (non-preemptive by default; haproxy-1 rejoins as backup)
ssh milad@192.168.122.14 "sudo systemctl start haproxy"
```

### Test 3 — Full chaos (master + one HAProxy down simultaneously)

```bash
ssh milad@192.168.100.11 "sudo systemctl stop redis-server" &
ssh milad@192.168.122.14 "sudo systemctl stop haproxy" &
wait

# Allow ~10 seconds for both failovers to complete, then verify
redis-cli -h 192.168.100.200 -a <password> ping
redis-cli -h 192.168.100.200 -a <password> get failover-test
```

---

## Read / Write traffic separation

HAProxy exposes two ports on the VIP:

| Port | Purpose | Routes to | Health check |
|------|---------|-----------|--------------|
| `6379` | **Writes** | Current master only | `role:master` |
| `6380` | **Reads** | Slaves only, round-robin | `role:slave` |

Point your application's Redis write client to `192.168.100.200:6379` and read client to `192.168.100.200:6380`.

After a Sentinel failover the promoted slave becomes master and its health check flips automatically — HAProxy re-routes writes to the new master and removes it from the read pool, all within 1–2 seconds.

```bash
# Test writes (goes to master)
redis-cli -h 192.168.100.200 -p 6379 -a <password> set foo bar

# Test reads (round-robins across slaves)
redis-cli -h 192.168.100.200 -p 6380 -a <password> get foo

# See which node each connection hits
redis-cli -h 192.168.100.200 -p 6380 -a <password> info server | grep tcp_port
```

---

## Ports reference

| Port | Service | Nodes |
|------|---------|-------|
| 6379 | Redis | redis-server-1/2/3 |
| 26379 | Redis Sentinel | redis-server-1/2/3 |
| 6379 | HAProxy → writes → master | haproxy-1/2 |
| 6380 | HAProxy → reads → slaves (round-robin) | haproxy-1/2 |
| 8585 | HAProxy stats UI | haproxy-1/2 |
