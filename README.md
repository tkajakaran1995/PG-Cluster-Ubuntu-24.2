# PostgreSQL 18 HA Cluster — Deployment Runbook

> **Community Edition** · 3-Node Co-located Cluster · Ubuntu 24.04 LTS · PostgreSQL 18.1  
> Stack: **Patroni + etcd + HAProxy** · OCI/on-prem variants included

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18.1-blue?logo=postgresql)](https://www.postgresql.org/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20LTS-orange?logo=ubuntu)](https://ubuntu.com/)
[![Patroni](https://img.shields.io/badge/Patroni-4.x-green)](https://github.com/patroni/patroni)
[![etcd](https://img.shields.io/badge/etcd-3.5.x-blue)](https://etcd.io/)
[![HAProxy](https://img.shields.io/badge/HAProxy-2.8.x%20LTS-red)](https://www.haproxy.org/)

---

## Table of Contents

- [What Changed vs. Ubuntu 22.04 / PG 16](#0-what-changed-vs-ubuntu-2204--pg-16)
- [Architecture Overview](#1-architecture-overview)
- [Base Preparation](#2-base-preparation-all-3-nodes)
- [Add the PGDG Repository](#3-add-the-pgdg-repository-all-3-nodes)
- [Install & Configure etcd](#4-install--configure-etcd-all-3-nodes)
- [Install PostgreSQL 18 + Patroni](#5-install-postgresql-18--patroni-all-3-nodes)
- [Configure Patroni](#6-configure-patroni-all-3-nodes)
- [Create the Patroni systemd Service](#7-create-the-patroni-systemd-service-all-3-nodes)
- [Install HAProxy](#8-install-haproxy-all-3-nodes)
- [Configure Keepalived VIP (On-Prem)](#9-configure-keepalived-vip-on-prem-only)
- [Oracle Cloud (OCI) Variant — NLB](#10-oracle-cloud-oci-variant--nlb-instead-of-keepalived)
- [Verify & Test Failover](#11-verify--test-failover)
- [On-Prem vs OCI — What Changes](#12-on-prem-vs-oci--what-changes)
- [Production Checklist](#13-production-checklist)
- [Appendix A — Migrating from PG 16](#appendix-a--migrating-an-existing-pg-16-cluster)
- [Appendix B — Troubleshooting](#appendix-b--troubleshooting-quick-reference)

---

## 0. What Changed vs. Ubuntu 22.04 / PG 16

> ⚠️ **Read this first** — five of these will break a copy-paste deployment from the old runbook.

| # | Area | 22.04 / PG 16 | 24.04 / PG 18.1 | Impact |
|---|------|---------------|-----------------|--------|
| 1 | PostgreSQL source | `apt install postgresql-16` (in Ubuntu repos) | PGDG repo required — Ubuntu 24.04 ships PG 16 only | 🔴 Blocker |
| 2 | Patroni install | `sudo pip3 install patroni` | Fails — PEP 668 "externally-managed-environment". Use a venv or PGDG package | 🔴 Blocker |
| 3 | Paths | `/…/16/…` | `/usr/lib/postgresql/18/bin`, `/var/lib/postgresql/18/main` | 🔴 Blocker |
| 4 | Auth method | `md5` in `pg_hba` | `scram-sha-256` — md5 is formally deprecated in PG 18 | 🟡 Should fix |
| 5 | HAProxy health check | `option httpchk GET /master` | `http-check send meth GET uri /primary` (HAProxy 2.8 syntax; `/master` is a deprecated alias) | 🟡 Fix |
| 6 | HAProxy mode | not set (inherits `mode http` from `defaults`) | `mode tcp` must be explicit | 🟠 Latent bug |
| 7 | Keepalived check | `killall -0 haproxy` | `psmisc` not installed on 24.04 minimal → use `pidof`; also `enable_script_security` | 🔴 Blocker |
| 8 | initdb checksums | — | `data-checksums` needed. Default ON in PG 18 (flag is now a no-op) | ⚪ Cosmetic |
| 9 | New PG 18 knobs | — | `io_method` / `io_workers` (async I/O), skip scan, UUIDv7, OAuth auth | 🔵 Tuning |
| 10 | etcd package | etcd-server 3.3.x | 3.4.x in Noble universe; upstream 3.5.x recommended | 🔵 Choose one |
| 11 | Firewall backend | `iptables-legacy` | `iptables-nft` (same CLI, `netfilter-persistent` still works) | ⚪ Note only |

---

## 1. Architecture Overview

Three identical nodes, each running the full stack: PostgreSQL 18, Patroni, etcd, HAProxy. Patroni handles automatic failover via etcd quorum; HAProxy routes writes to the current primary and load-balances reads across replicas.

```
Client apps
|
[ VIP (Keepalived) / OCI NLB ]
|
+----------+----------+
|          |          |
+--v--+  +--v--+  +--v--+
| n1  |  | n2  |  | n3  |
+-----+  +-----+  +-----+

Each node = PostgreSQL 18 + Patroni + etcd + HAProxy
(etcd 3-node quorum across n1/n2/n3)
```

### Node Reference Values

| Node | IP | Patroni name | Keepalived state / priority |
|------|----|--------------|-----------------------------|
| n1 | `10.0.0.1` | `n1` | MASTER / 101 |
| n2 | `10.0.0.2` | `n2` | BACKUP / 100 |
| n3 | `10.0.0.3` | `n3` | BACKUP / 99 |

**Virtual IP (on-prem):** `10.0.0.100` · **Subnet:** `10.0.0.0/24`

### Version Matrix

| Component | Version | Source |
|-----------|---------|--------|
| OS | Ubuntu 24.04 LTS (24.04.2+), kernel 6.8 | Canonical |
| PostgreSQL | 18.1 | apt.postgresql.org (PGDG) |
| Patroni | 4.x | PyPI (venv) or PGDG |
| etcd | 3.5.x | upstream tarball (or Noble `etcd-server` 3.4.x) |
| HAProxy | 2.8.x LTS | Ubuntu 24.04 |
| Keepalived | 2.2.x | Ubuntu 24.04 (on-prem only) |
| Python | 3.12 | Ubuntu 24.04 |

### Minimal VM Specification (per node)

| Resource | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| vCPU | 2 | 4–8 | PG 18 async I/O adds `io_workers` (3 by default) |
| RAM | 8 GB | 16 GB+ | `shared_buffers` ≈ 25% of RAM |
| OS disk | 40 GB | 50 GB | 24.04 footprint is larger than 22.04 |
| Data disk | 50 GB SSD | 100 GB+ NVMe SSD | etcd is fsync-sensitive |
| etcd disk | shares data disk | separate volume | Isolate etcd fsync from PG I/O in production |
| Count | 3 nodes | 3 or 5 | Odd number required for etcd quorum |

> **Why 3 nodes:** etcd requires an odd-numbered quorum. Two nodes cannot perform safe automatic failover — 3 is the minimum for true HA.

---

## 2. Base Preparation (All 3 Nodes)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg ca-certificates psmisc chrony
sudo timedatectl set-ntp true
timedatectl status   # confirm "System clock synchronized: yes"

# Set hostname per node (n1 / n2 / n3)
sudo hostnamectl set-hostname n1

cat <<EOF | sudo tee -a /etc/hosts
10.0.0.1 n1
10.0.0.2 n2
10.0.0.3 n3
EOF
```

> ⏰ **Clock skew breaks etcd leases — do not skip NTP.**  
> `psmisc` is installed here because Keepalived's health check needs it (24.04 minimal images omit it).

### Kernel Tuning (optional but recommended)

```bash
cat <<EOF | sudo tee /etc/sysctl.d/60-postgres.conf
vm.swappiness = 1
vm.overcommit_memory = 2
vm.overcommit_ratio = 90
net.ipv4.tcp_keepalive_time = 300
EOF
sudo sysctl --system
```

---

## 3. Add the PGDG Repository (All 3 Nodes)

Ubuntu 24.04's own archive only carries PostgreSQL 16. PostgreSQL 18.1 comes from `apt.postgresql.org`.

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
sudo apt update
apt-cache madison postgresql-18   # confirm 18.1 is available
```

> The helper script installs the signing key and writes `/etc/apt/sources.list.d/pgdg.list` with `signed-by=` — do not use the old `apt-key` method.

---

## 4. Install & Configure etcd (All 3 Nodes)

Pick **one** option. Option A is deterministic and gives you a current release; Option B is faster but tracks etcd 3.4.x from universe.

### Option A — Upstream Binaries (Recommended)

```bash
ETCD_VER=v3.5.21   # use the current 3.5.x patch release
cd /tmp
curl -fsSL -o etcd.tar.gz \
  https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd.tar.gz
sudo install -m 0755 etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/
sudo install -m 0755 etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/

sudo useradd --system --home /var/lib/etcd --shell /usr/sbin/nologin etcd || true
sudo mkdir -p /var/lib/etcd /etc/etcd
sudo chown -R etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
```

Create `/etc/etcd/etcd.conf.yml` — **change `name` and all IPs per node:**

```yaml
name: n1
data-dir: /var/lib/etcd
listen-peer-urls: http://10.0.0.1:2380
listen-client-urls: http://10.0.0.1:2379,http://127.0.0.1:2379
initial-advertise-peer-urls: http://10.0.0.1:2380
advertise-client-urls: http://10.0.0.1:2379
initial-cluster: n1=http://10.0.0.1:2380,n2=http://10.0.0.2:2380,n3=http://10.0.0.3:2380
initial-cluster-token: pg-etcd
initial-cluster-state: new
enable-v2: false
auto-compaction-mode: periodic
auto-compaction-retention: "1"
```

Create `/etc/systemd/system/etcd.service`:

```ini
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf.yml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
```

### Option B — Distro Package

```bash
sudo add-apt-repository -y universe
sudo apt update
sudo apt install -y etcd-server etcd-client
# Edit /etc/default/etcd with ETCD_NAME, ETCD_LISTEN_*, ETCD_INITIAL_CLUSTER, etc.
sudo systemctl enable --now etcd
```

### Verify (Either Option)

```bash
export ETCDCTL_API=3
etcdctl --endpoints=http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379 \
  member list -w table

etcdctl --endpoints=http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379 \
  endpoint health -w table
```

> ✅ All three members must be healthy before you start Patroni.

---

## 5. Install PostgreSQL 18 + Patroni (All 3 Nodes)

```bash
sudo apt install -y postgresql-18 postgresql-contrib-18 \
  python3-psycopg2 python3-venv python3-pip

# To pin the exact minor release:
sudo apt install -y \
  postgresql-18=18.1-1.pgdg24.04+1 \
  postgresql-client-18=18.1-1.pgdg24.04+1
```

### Install Patroni into a venv (PEP 668 workaround)

> ❌ `sudo pip3 install patroni` fails on Ubuntu 24.04 with `error: externally-managed-environment`.  
> Do **not** use `--break-system-packages`. Use an isolated venv instead:

```bash
sudo python3 -m venv --system-site-packages /opt/patroni
sudo /opt/patroni/bin/pip install --upgrade pip wheel
sudo /opt/patroni/bin/pip install "patroni[etcd3]"

# Make patronictl available on PATH
sudo ln -sf /opt/patroni/bin/patronictl /usr/local/bin/patronictl
/opt/patroni/bin/patroni --version
```

### Hand PostgreSQL Over to Patroni

```bash
# Patroni manages PostgreSQL — disable the packaged service and cluster
sudo systemctl disable --now postgresql
sudo pg_dropcluster --stop 18 main 2>/dev/null || true

# Data dir must be empty for first bootstrap
sudo rm -rf /var/lib/postgresql/18/main
sudo mkdir -p /var/lib/postgresql/18/main /etc/patroni
sudo chown -R postgres:postgres /var/lib/postgresql/18 /etc/patroni
sudo chmod 700 /var/lib/postgresql/18/main
```

> `pg_dropcluster` is the clean way to remove the auto-created cluster on Debian/Ubuntu — it also removes `/etc/postgresql/18/main`, which otherwise confuses `pg_ctlcluster`.

---

## 6. Configure Patroni (All 3 Nodes)

Create `/etc/patroni/patroni.yml` — **change `name` and the 3 IPs per node:**

```yaml
scope: pg-cluster
namespace: /service/
name: n1

restapi:
  listen: 10.0.0.1:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: false   # set true if you need zero-data-loss failover
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 1GB
        wal_compression: lz4
        password_encryption: scram-sha-256
        # --- PostgreSQL 18 asynchronous I/O ---
        io_method: worker       # worker | sync | io_uring
        io_workers: 3
        # --- sizing: adjust to the VM ---
        shared_buffers: 4GB          # ~25% of RAM
        effective_cache_size: 12GB   # ~75% of RAM
        max_connections: 200
        checkpoint_timeout: 15min
        max_wal_size: 4GB
        log_min_duration_statement: 1000
  initdb:
    - encoding: UTF8
    - locale-provider: builtin
    - builtin-locale: C.UTF-8
    - data-checksums
  pg_hba:
    - local all all peer
    - host replication replicator 10.0.0.0/24 scram-sha-256
    - host all all 10.0.0.0/24 scram-sha-256
    - host all all 127.0.0.1/32 scram-sha-256

postgresql:
  listen: 10.0.0.1:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/18/main
  bin_dir: /usr/lib/postgresql/18/bin
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: CHANGE_ME_repl
    superuser:
      username: postgres
      password: CHANGE_ME_super
    rewind:
      username: rewind_user
      password: CHANGE_ME_rewind

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

### PG 18 Notes

- **`scram-sha-256`** replaces `md5`. MD5 password auth is deprecated in PG 18.
- **`data-checksums`** is the `initdb` default in PG 18. Leaving the flag in is harmless and self-documenting.
- **`io_method: worker`** is new in PG 18. Start with `worker`, benchmark before switching to `io_uring`.
- **`locale-provider: builtin`** with `C.UTF-8` is stable across glibc upgrades and avoids index corruption from collation drift.
- **Rewind user:** a dedicated role for `pg_rewind` rather than reusing the superuser. Patroni creates it during bootstrap.

---

## 7. Create the Patroni systemd Service (All 3 Nodes)

Create `/etc/systemd/system/patroni.service`:

```ini
[Unit]
Description=Patroni PostgreSQL HA
After=network-online.target etcd.service
Wants=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/opt/patroni/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
KillMode=process
TimeoutSec=30
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> ⚠️ `ExecStart` path is `/opt/patroni/bin/patroni`, not `/usr/local/bin/patroni`.

**Start order matters:** start n1 first (it bootstraps and becomes primary), wait for it to reach `running`, then n2, then n3.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now patroni

# On n1, watch the bootstrap
sudo journalctl -u patroni -f
patronictl -c /etc/patroni/patroni.yml list
```

Expected output — one Leader, two Replicas, all `running`:

```
+ Cluster: pg-cluster (7xxxxxxxxxxxxxxxxxx) -----+----+-----------+
| Member | Host     | Role    | State     | TL | Lag in MB |
+--------+----------+---------+-----------+----+-----------+
| n1     | 10.0.0.1 | Leader  | running   |  1 |           |
| n2     | 10.0.0.2 | Replica | streaming |  1 |         0 |
| n3     | 10.0.0.3 | Replica | streaming |  1 |         0 |
+--------+----------+---------+-----------+----+-----------+
```

> Patroni 4 reports replica state as `streaming` (rather than `running`) once WAL is flowing — that is the healthy state.

---

## 8. Install HAProxy (All 3 Nodes)

```bash
sudo apt install -y haproxy
haproxy -v   # expect 2.8.x
```

Append to `/etc/haproxy/haproxy.cfg` (identical on all 3 nodes):

```
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats refresh 5s

listen postgres-write
    bind *:5000
    mode tcp
    option httpchk
    http-check send meth GET uri /primary
    http-check expect status 200
    timeout client 30m
    timeout server 30m
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server n1 10.0.0.1:5432 maxconn 100 check port 8008
    server n2 10.0.0.2:5432 maxconn 100 check port 8008
    server n3 10.0.0.3:5432 maxconn 100 check port 8008

listen postgres-read
    bind *:5001
    mode tcp
    option httpchk
    http-check send meth GET uri /replica
    http-check expect status 200
    balance roundrobin
    timeout client 30m
    timeout server 30m
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server n1 10.0.0.1:5432 maxconn 100 check port 8008
    server n2 10.0.0.2:5432 maxconn 100 check port 8008
    server n3 10.0.0.3:5432 maxconn 100 check port 8008
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # validate before restarting
sudo systemctl enable --now haproxy
sudo systemctl restart haproxy
```

Three deliberate changes from the old config:

1. `mode tcp` is now explicit — without it the listener inherits `mode http` from defaults and mangles the PostgreSQL wire protocol.
2. `option httpchk GET /primary` one-liner is deprecated in HAProxy 2.2+; the `http-check send meth GET uri …` form is current.
3. `timeout client/server 30m` overrides the 50-second default, which would otherwise silently drop idle database sessions.

> `/primary` is the current Patroni endpoint name; `/master` still works as a deprecated alias.

---

## 9. Configure Keepalived VIP (On-Prem Only)

> ⛅ **Skip this section on Oracle Cloud (OCI)** — OCI's virtual network blocks VRRP / gratuitous ARP for floating IPs. Use the OCI NLB in [Section 10](#10-oracle-cloud-oci-variant--nlb-instead-of-keepalived) instead.

```bash
sudo apt install -y keepalived
ip -br a   # confirm the interface name (often ens3 / enp0s3 on 24.04)
```

Create `/etc/keepalived/keepalived.conf` — **change `state` and `priority` per node:**

```
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
    script "/usr/bin/pidof haproxy"
    interval 2
    weight 2
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    interface ens3           # verify with: ip -br a
    state MASTER             # BACKUP on n2 / n3
    virtual_router_id 51
    priority 101             # 100 on n2, 99 on n3
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass CHANGE_ME_vip
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        chk_haproxy
    }
}
```

```bash
sudo systemctl enable --now keepalived
ip a | grep 10.0.0.100   # VIP should appear on n1
```

> **Why the check script changed:** `killall -0 haproxy` requires `psmisc`, which is absent on 24.04 minimal images. `pidof` is in `procps`, which is always installed. Keepalived 2.2 also refuses to run unprivileged scripts unless `enable_script_security` / `script_user` are set explicitly.

---

## 10. Oracle Cloud (OCI) Variant — NLB instead of Keepalived

On OCI, replace [Section 9](#9-configure-keepalived-vip-on-prem-only) with an OCI Network Load Balancer. Sections 1–8 are unchanged.

### 10.1 — Skip Keepalived

Do not install or configure Keepalived. The NLB provides the stable entry point.

### 10.2 — Open Ports in the VCN

OCI Console → **Networking → VCN → Security Lists** (or attach an NSG). Add ingress rules:

| Port | Purpose | Source |
|------|---------|--------|
| 5432 | PostgreSQL | VCN CIDR `10.0.0.0/24` |
| 8008 | Patroni REST (health) | VCN CIDR `10.0.0.0/24` |
| 2379, 2380 | etcd | VCN CIDR `10.0.0.0/24` |
| 5000, 5001 | HAProxy write / read | VCN CIDR + app CIDR |
| 7000 | HAProxy stats | admin CIDR only (optional) |

> NSGs are preferred over Security Lists — they attach to VNICs rather than the whole subnet and can reference other NSGs as sources.

### 10.3 — Open Ports on Each Ubuntu Node

OCI Ubuntu images ship with iptables rules that block traffic even after Security Lists allow it:

```bash
sudo iptables -L INPUT -n --line-numbers   # note the trailing REJECT rule
sudo iptables -I INPUT -p tcp -m multiport \
  --dports 5432,8008,2379,2380,5000,5001 -j ACCEPT
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

> `-I` inserts at the **top** of the chain, ahead of OCI's default REJECT — that ordering is the whole point.  
> If you prefer `ufw`, disable the OCI iptables rules first; running both is a common source of partial connectivity issues.

### 10.4 — Create the Network Load Balancer

OCI Console → **Networking → Load Balancers → Network Load Balancer → Create**. Place it in the same VCN (private subnet recommended). Create two listeners + two backend sets:

| Path | Listener | Backend set | Health check |
|------|----------|-------------|--------------|
| Write (to primary) | TCP 5000 | n1/n2/n3 on port 5000 | TCP 5000 |
| Read (replicas) | TCP 5001 | n1/n2/n3 on port 5001 | TCP 5001 |

> The NLB forwards to HAProxy on each node. HAProxy does the real primary/replica routing using Patroni's health endpoint (port 8008). The NLB health check is plain TCP — it only confirms HAProxy is alive.  
> **Source IP preservation:** OCI NLB preserves the client source IP by default. Your `pg_hba` rules must allow the application CIDR, not the NLB's address.

### 10.5 — Connect via the NLB IP

```bash
# Writes
psql -h <NLB-IP> -p 5000 -U postgres -c "SELECT inet_server_addr();"

# Reads
psql -h <NLB-IP> -p 5001 -U postgres -c "SELECT pg_is_in_recovery();"
```

---

## 11. Verify & Test Failover

```bash
# Cluster state
patronictl -c /etc/patroni/patroni.yml list

# Writes — always routed to the current primary
psql -h 10.0.0.100 -p 5000 -U postgres -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# Reads — load-balanced across replicas
psql -h 10.0.0.100 -p 5001 -U postgres -c "SELECT inet_server_addr(), pg_is_in_recovery();"

# Confirm version and PG 18 async I/O
psql -h 10.0.0.100 -p 5000 -U postgres -c "SELECT version();"
psql -h 10.0.0.100 -p 5000 -U postgres -c "SHOW io_method; SHOW io_workers;"

# Replication health
psql -h 10.0.0.100 -p 5000 -U postgres \
  -c "SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"

# Controlled failover
patronictl -c /etc/patroni/patroni.yml switchover

# Ungraceful test: kill the primary's Patroni and watch promotion (~ttl seconds)
sudo systemctl stop patroni   # on the current leader
watch -n1 'patronictl -c /etc/patroni/patroni.yml list'
```

**Success criteria:** after switchover, a former replica shows `Leader / running`, the old primary rejoins as a replica on the same timeline (`pg_rewind` may run), and writes through port 5000 continue to reach the new primary with no client reconfiguration. Unplanned failover should complete within roughly `ttl` (30s) plus HAProxy's `inter × fall` (9s).

---

## 12. On-Prem vs OCI — What Changes

| | On-prem / L2 network | Oracle Cloud (OCI) |
|---|---|---|
| **Stable entry point** | Keepalived VIP (`10.0.0.100`) | OCI NLB private IP |
| **Section 9 (Keepalived)** | Configure as shown | Skip — create NLB instead |
| **Failover routing** | HAProxy + Patroni | HAProxy + Patroni (unchanged) |
| **Firewall** | Optional | VCN Security List/NSG + local iptables (required) |
| **Sections 1–8** | As written | Identical |

---

## 13. Production Checklist

- [ ] **Secrets:** replace `CHANGE_ME_repl`, `CHANGE_ME_super`, `CHANGE_ME_rewind`, `CHANGE_ME_vip` with strong generated secrets. Keep `patroni.yml` at mode `0600`, owned by `postgres`.
- [ ] **Access control:** tighten `pg_hba` and firewall rules to your actual application subnet only. Do not leave `host all all 10.0.0.0/24` open.
- [ ] **Authentication:** `scram-sha-256` everywhere. PG 18 also adds OAuth 2.0 bearer-token auth (`oauth` in `pg_hba`) if you have an existing IdP.
- [ ] **Backups:** set up pgBackRest or Barman separately — HA is not backup. Replication does not protect against accidental deletes or corruption. pgBackRest integrates with Patroni via `create_replica_methods`.
- [ ] **TLS:** enable TLS for client connections (`ssl = on`, `ssl_cert_file`, `ssl_key_file`) and for replication traffic. Consider TLS between etcd peers too (`etcd3.protocol: https` plus cert config in `patroni.yml`).
- [ ] **Tuning:** `shared_buffers` ≈ 25% RAM, `effective_cache_size` ≈ 75% RAM, `max_connections` sized to your pooler. Consider PgBouncer in front of HAProxy for high connection counts.
- [ ] **PG 18 async I/O:** leave `io_method = worker` initially; benchmark `io_uring` on NVMe before adopting. Watch `pg_aios` and `pg_stat_io`.
- [ ] **Monitoring:** Prometheus + `postgres_exporter`, plus Patroni's own `/metrics` endpoint on port 8008 and etcd's `/metrics` on 2379. Alert on: leader changes, replica lag, etcd quorum loss, `pg_stat_replication` count < 2.
- [ ] **etcd disk:** keep etcd on low-latency SSD, ideally a separate volume from PGDATA. Alert on `etcd_disk_wal_fsync_duration_seconds` p99 > 25ms.
- [ ] **Pin versions:** prevent unattended-upgrades from restarting the database under you:
  ```bash
  sudo apt-mark hold postgresql-18 postgresql-client-18 haproxy
  ```
  Patch deliberately: replicas first, then `patronictl switchover`, then the old primary.
- [ ] **Watchdog:** for true split-brain protection, enable the `softdog` watchdog device and Patroni's `watchdog:` section on all nodes.
- [ ] **Synchronous mode:** if losing recent transactions is unacceptable, set `synchronous_mode: true` in the DCS config (costs write latency).

---

## Appendix A — Migrating an Existing PG 16 Cluster

Two options, depending on downtime tolerance.

### Option 1: In-place `pg_upgrade` (minutes of downtime, same hosts)

```bash
# Stop Patroni everywhere, keep only the ex-primary's data
sudo systemctl stop patroni

sudo -u postgres /usr/lib/postgresql/18/bin/pg_upgrade \
  --old-datadir /var/lib/postgresql/16/main \
  --new-datadir /var/lib/postgresql/18/main \
  --old-bindir  /usr/lib/postgresql/16/bin \
  --new-bindir  /usr/lib/postgresql/18/bin \
  --jobs 4 --swap --check   # drop --check to run for real
```

> PG 18's `--swap` moves data directories instead of copying/linking — substantially faster on large clusters.  
> Replicas must then be re-bootstrapped from the upgraded primary (`patronictl reinit`).  
> Run `vacuumdb --all --analyze-in-stages` immediately after — planner statistics are not carried over.

### Option 2: Logical Replication (near-zero downtime, new hosts)

Build the PG 18 cluster from this runbook alongside the old one, create a publication on PG 16 and a subscription on PG 18, wait for catch-up, then cut the application over. Requires primary keys on all replicated tables and manual handling of sequences.

---

## Appendix B — Troubleshooting Quick Reference

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `error: externally-managed-environment` | PEP 668 on Ubuntu 24.04 | Use the `/opt/patroni` venv ([Section 5](#5-install-postgresql-18--patroni-all-3-nodes)) |
| Patroni starts, exits immediately | Bad YAML indentation or wrong `bin_dir` | `journalctl -u patroni -n 50`; verify `/usr/lib/postgresql/18/bin` exists |
| All members show as replicas, no leader | etcd quorum not formed | `etcdctl endpoint health`; check ports 2379/2380 |
| `psql` through HAProxy hangs or returns garbage | `mode tcp` missing on the listener | Add `mode tcp` ([Section 8](#8-install-haproxy-all-3-nodes)) |
| Sessions drop after ~50s idle | HAProxy default timeouts | `timeout client/server 30m` |
| HAProxy marks all backends DOWN | Port 8008 blocked, or checking `/master` on a strict setup | Open 8008; use `/primary` |
| VIP present on two nodes | VRRP blocked (cloud network) or `virtual_router_id` collision | Use OCI NLB; make the router ID unique per L2 segment |
| Replica stuck in `start failed` after failover | Timeline divergence, `pg_rewind` unavailable | Verify `use_pg_rewind: true` and the rewind user; else `patronictl reinit <cluster> <member>` |
| `password authentication failed` after upgrade | md5 hashes vs SCRAM | `ALTER USER … PASSWORD …` with `password_encryption = scram-sha-256` |

---

## License

This runbook is released under the [MIT License](LICENSE).

---

*Document version 2.0 — Ubuntu 24.04 LTS / PostgreSQL 18.1. Supersedes v1.0 (Ubuntu 22.04 / PostgreSQL 16).*
