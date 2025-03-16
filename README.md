# Multiple Tailscale With Load Balancing Caddy

## VM Creation

Create the Rocky Linux 9 VM from a `cloud-init` template.

| Parameter  | Value |
| ---------- | ---- |
| Memory     | 2048 MiB
| Swap       | 512 MiB
| Cores      | 2
| Root Disk  | 10GB (template default)

## VM Configuration

Remove package not needed

    dnf remove -y cockpit-bridge cockpit-system cockpit-ws
    dnf update -y
    dnf install -y curl less vim git

Edit `/etc/systemd/journald.conf` and set `SystemMaxUse=400M` to limit logging as we provisioned a small disk.

    sed -i "/SystemMaxUse=/c\SystemMaxUse=400M" /etc/systemd/journald.conf
    systemctl restart systemd-journald
    systemctl status systemd-journald 

## Cloud-Init

To disable cloud-init, create the empty file `/etc/cloud/cloud-init.disabled`

## Dotfiles

Clone the repo

    cd ~
    git clone https://github.com/adilinden-oss/etc-docker-dotfiles.git .etc-dotfiles

Initialize dotfiles

    cd .etc-dotfiles
    ./makelinks.sh

Add this to `~/.gitconfig`:

```
[include]
    path = ~/.gitconfig_add
```

Adjust `.bashrc` by adding this, if necessary

    if [ -f ~/.bashrc_add ]; then . ~/.bashrc_add; fi

## Docker

Install Docker.

    dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    dnf install -y docker-ce docker-ce-cli containerd.io

Configure `/etc/docker/daemon.json` to use the system `journald` logging mechanism.

```
{
  "log-driver": "journald"
}
```

Enable and start `docker`

    systemctl enable docker
    systemctl restart docker
    systemctl status docker

## Services

Quick start

    git clone https://github.com/adilinden-oss/docker-my-mgmt.git

Copy `example.env` to `.env` and adjust as needed.

### Proxmox VE

Create this `docker-compose.yml`

```
services:

  ts-pve:
    image: tailscale/tailscale
    container_name: ts-pve
    hostname: ${TS_PVE_HOSTNAME}
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_AUTH_KEY=${TS_PVE_AUTH_KEY}
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ${PWD}/volumes/ts-pve/lib:/var/lib
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-pve/run:/tmp
    cap_add:
      - NET_ADMIN
    restart: unless-stopped

  caddy-pve:
    image: caddy:latest
    container_name: caddy-pve
    depends_on:
      - ts-pve
    environment:
      - CADDY_CERT_DOMAIN=${CADDY_PVE_CERT_DOMAIN}
      - CADDY_UPSTREAM1=${CADDY_PVE_UPSTREAM1}
      - CADDY_UPSTREAM2=${CADDY_PVE_UPSTREAM2}
      - CADDY_UPSTREAM3=${CADDY_PVE_UPSTREAM3}
    volumes:
      - ${PWD}/config/caddy-pve/Caddyfile:/etc/caddy/Caddyfile
      - ${PWD}/volumes/caddy-pve/data:/data
      - ${PWD}/volumes/caddy-pve/config:/config
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-pve/run:/var/run/tailscale
    network_mode: service:ts-pve
    restart: unless-stopped
```

This is an example `.env` file for this service

```
# Tailscale (Proxmox Cluster)
TS_PVE_HOSTNAME=mgmt-pv
TS_PVE_AUTH_KEY=tskey-auth-1234567890

# Caddy (Proxmox Cluster)
CADDY_PVE_CERT_DOMAIN=mgmt-pv.my_tailnet_name.ts.net
CADDY_PVE_UPSTREAM1=https://192.168.99.51:8006
CADDY_PVE_UPSTREAM2=https://192.168.99.52:8006
CADDY_PVE_UPSTREAM3=https://192.168.99.53:8006
```

Create `config/caddy-pve/Caddyfile` configuration file

```
{$CADDY_CERT_DOMAIN} {
    reverse_proxy * {
        to {$CADDY_UPSTREAM1}
        to {$CADDY_UPSTREAM2}
        to {$CADDY_UPSTREAM3}

        lb_policy ip_hash     # Makes backend sticky based on client ip
        lb_try_duration 10s
        lb_try_interval 250ms

        health_uri /          # Backend health check path
        # health_port 80      # Default same as backend port
        health_interval 30s
        health_timeout 5s
        health_status 200

        transport http {
            tls
	        tls_insecure_skip_verify
	        read_buffer 8192
        }
    }
}
```

### Firewall

Create this `docker-compose.yml`

```
services:

  ts-fw:
    image: tailscale/tailscale
    container_name: ts-fw
    hostname: ${TS_FW_HOSTNAME}
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_AUTH_KEY=${TS_FW_AUTH_KEY}
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ${PWD}/volumes/ts-fw/lib:/var/lib
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-fw/run:/tmp
    cap_add:
      - NET_ADMIN
    restart: unless-stopped

  caddy-fw:
    image: caddy:latest
    container_name: caddy-fw
    depends_on:
      - ts-fw
    environment:
      - CADDY_CERT_DOMAIN=${CADDY_FW_CERT_DOMAIN}
      - CADDY_UPSTREAM=${CADDY_FW_UPSTREAM}
    volumes:
      - ${PWD}/config/caddy-fw/Caddyfile:/etc/caddy/Caddyfile
      - ${PWD}/volumes/caddy-fw/data:/data
      - ${PWD}/volumes/caddy-fw/config:/config
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-fw/run:/var/run/tailscale
    network_mode: service:ts-fw
    restart: unless-stopped
```

This is an example `.env` file for this service

```
# Tailscale (Firewall)
TS_FW_HOSTNAME=mgmt-fw
TS_FW_AUTH_KEY=tskey-1234567890

# Caddy (Firewall)
CADDY_FW_CERT_DOMAIN=mgmt-fw.my_tailnet_name.ts.net
CADDY_FW_UPSTREAM=https://192.168.99.1:443
```

Create `config/caddy-fw/Caddyfile` configuration file

```
{$CADDY_CERT_DOMAIN} {
    reverse_proxy {$CADDY_UPSTREAM} {
        transport http {
            tls
            tls_insecure_skip_verify
            read_buffer 8192
        }
    }
}
```

### Proxmox Backup Server

Create this `docker-compose.yml`

```
services:

  ts-pbs:
    image: tailscale/tailscale
    container_name: ts-pbs
    hostname: ${TS_PBS_HOSTNAME}
    environment:
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_AUTH_KEY=${TS_PBS_AUTH_KEY}
    volumes:
      - /dev/net/tun:/dev/net/tun
      - ${PWD}/volumes/ts-pbs/lib:/var/lib
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-pbs/run:/tmp
    cap_add:
      - NET_ADMIN
    restart: unless-stopped

  caddy-pbs:
    image: caddy:latest
    container_name: caddy-pbs
    depends_on:
      - ts-pbs
    environment:
      - CADDY_CERT_DOMAIN=${CADDY_PBS_CERT_DOMAIN}
      - CADDY_UPSTREAM=${CADDY_PBS_UPSTREAM}
    volumes:
      - ${PWD}/config/caddy-pbs/Caddyfile:/etc/caddy/Caddyfile
      - ${PWD}/volumes/caddy-pbs/data:/data
      - ${PWD}/volumes/caddy-pbs/config:/config
      # tailscaled.sock shared with caddy
      - ${PWD}/volumes/ts-pbs/run:/var/run/tailscale
    network_mode: service:ts-pbs
    restart: unless-stopped
```

This is an example `.env` file for this service

```
# Tailscale (Proxmox Backup Server)
TS_PBS_HOSTNAME=mgmt-fw
TS_PBS_AUTH_KEY=tskey-1234567890

# Caddy (Proxmox Backup Server)
CADDY_PBS_CERT_DOMAIN=mgmt-fw.my_tailnet_name.ts.net
CADDY_PBS_UPSTREAM=https://192.168.99.75:443
```

Create `config/caddy-fw/Caddyfile` configuration file

```
{$CADDY_CERT_DOMAIN} {
    reverse_proxy {$CADDY_UPSTREAM} {
        transport http {
            tls
            tls_insecure_skip_verify
            read_buffer 8192
        }
    }
}
```
