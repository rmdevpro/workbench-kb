# RUN-002: Host Setup

Bringing a new bare metal host to the standard defined in `INF-001-bare-metal-configuration.md`.

---

## 1. OS Install

- Ubuntu 24.04 LTS (Noble) from USB/PXE
- Single partition for root, or LVM — `/srv` and `/mnt/storage` will be added later as separate mount points
- Create user `aristotle9` during install, UID 1000

## 2. Base Packages

```bash
sudo apt-get update
sudo apt-get install -y \
  openssh-server sshpass \
  docker-ce docker-compose-plugin \
  nfs-common lsyncd
```

GPU nodes additionally need:
```bash
sudo apt-get install -y nvidia-driver-<version> nvidia-container-toolkit
```

Storage nodes additionally need:
```bash
sudo apt-get install -y zfsutils-linux nfs-kernel-server
```

## 3. Network

Set static IP in `/etc/netplan/<config>.yaml`:
```yaml
network:
  version: 2
  ethernets:
    <iface>:
      addresses: [192.168.1.XXX/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.110, 192.168.1.120, 1.1.1.1]
```

> The cluster runs CoreDNS on irina and m5 for `.lan` resolution. `1.1.1.1` is retained
> as a fallback for external DNS during CoreDNS maintenance windows. The Comcast router
> (`192.168.1.1`) is no longer used as a resolver.
>
> **Before CoreDNS is deployed** on a new host, you must also disable the systemd-resolved
> stub listener so CoreDNS can bind to port 53:
> ```bash
> sudo tee -a /etc/systemd/resolved.conf <<'EOF'
> [Resolve]
> DNSStubListener=no
> EOF
> sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
> sudo systemctl restart systemd-resolved
> ```

For bonded interfaces, see INF-001 §Network.

Apply: `sudo netplan apply`

## 4. SSH

Enable password auth in `/etc/ssh/sshd_config`:
```
PasswordAuthentication yes
PermitRootLogin prohibit-password
```

Generate root ed25519 key:
```bash
sudo ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""
```

Copy to peer hosts (run from this host, for each peer):
```bash
sudo ssh-copy-id -i /root/.ssh/id_ed25519.pub root@<peer-ip>
```

## 5. Identity Sync (`lsyncd`)

Create `/etc/lsyncd/lsyncd.conf.lua` — see INF-001 §Identity Sync for the full config.

Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lsyncd-identity.service
```

## 6. Docker

Add user to docker group:
```bash
sudo usermod -aG docker aristotle9
```

Log out and back in. Verify:
```bash
docker ps   # must work without sudo
```

## 7. Docker Swarm

**First host in the cluster:**
```bash
docker swarm init --advertise-addr <this-host-ip>
docker network create --driver overlay --attachable joshua-net
```

**Subsequent hosts (join as manager):**
```bash
# On existing swarm leader, get token:
docker swarm join-token manager

# On new host:
docker swarm join --token <manager-token> <leader-ip>:2377
```

## 8. Storage Setup

### `/srv/` (all hosts)

**On a host with its own pool (irina-style):**
```bash
sudo zfs create -o mountpoint=/srv <pool>/srv
sudo chmod 777 /srv
```

**On a host with a dedicated drive (M5-style):**
```bash
# After partitioning/formatting the drive:
sudo mkdir -p /srv
sudo chmod 777 /srv
# Add to /etc/fstab:
#   UUID=<uuid>  /srv  <fs>  defaults  0  2
sudo mount -a
```

### `/mnt/storage/` (non-storage hosts only)

```bash
sudo mkdir -p /mnt/storage
```

Add to `/etc/fstab`:
```
192.168.1.110:/mnt/storage /mnt/storage nfs defaults 0 0
```

```bash
sudo mount -a
```

### NFS export (storage node only)

Add to `/etc/exports`:
```
/mnt/storage 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
```

```bash
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
```

## 9. Seed compose manifests

The host needs the compose override files from the Admin repo placed at `/srv/.admin/<service>/docker-compose.override.yml` for each service it runs. The full Admin repo does not live on the host — only the per-service override files.

From a developer workstation or the workbench:

```bash
# For each service this host runs:
scp Admin/compose/<host>/<service>/docker-compose.override.yml \
    aristotle9@<host>:/srv/.admin/<service>/docker-compose.override.yml
```

The companion file (`docker-compose.yml` from the service's public repo) is written alongside it during the first deployment of each service — see RUN-001 step 5.

## 10. Verify

- `docker node ls` (on any swarm manager) shows this host
- `ls /srv` exists and is 777
- `ls /mnt/storage` shows purpose-organized directories
- `docker pull irina:5000/<test-image>` succeeds (registry reachable)
