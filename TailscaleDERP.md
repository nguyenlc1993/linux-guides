# Tailscale DERP relay, peer relay, and exit node on a VPS

This guide is written in ASD-STE100 Simplified Technical English.

## 1. Purpose and scope

This guide shows how to install a self-hosted Tailscale DERP server on a VPS.
The same server also operates as a peer relay and as an exit node.

A DERP server relays encrypted traffic between two Tailscale nodes.
The nodes use the DERP server when they cannot make a direct connection.
A self-hosted DERP server decreases the latency for nodes that are near to it.

The guide includes the preparation of the operating system.
It also includes the hardening steps, the monitoring, and the tests.

For the DNS server on the same machine, refer to `AdGuardHome.md`.

### 1.1. Values in this guide

Replace these values with your own values.

| Value | Example in this guide |
| --- | --- |
| Public IPv4 address | `203.0.113.10` |
| DERP host name | `derp.example.com` |
| Tailnet IPv4 address | `100.x.y.z` |
| Administrator user | `ubuntu` |
| Tailscale tag | `tag:relay` |

## 2. Prerequisites

Before you start, make sure that these conditions are true.

- The server has Ubuntu 24.04 LTS.
- The server has a public IPv4 address.
- You have root access with an SSH key.
- You have a Tailscale account and an administrator access to the tailnet.
- You control a domain name.
- The server has a minimum of 1 GB of RAM and 10 GB of disk space.

### 2.1. Make the DNS record

Make an `A` record that points to the public IPv4 address of the server.

If the server has an IPv6 address, also make an `AAAA` record.
Let's Encrypt prefers IPv6. If the `AAAA` record is incorrect, the
certificate request fails.

If you use Cloudflare, set the record to *DNS only*. Do not use the proxy.
The proxy terminates TLS. The DERP protocol does not operate through it.

Wait until the record resolves.

```bash
dig +short A derp.example.com
```

## 3. Prepare the server

### 3.1. Add the swap space

The Go compiler needs more memory than a 1 GB server has.
Add the swap space before you build the software.

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

Make sure that the swap space is available.

```bash
swapon --show
```

### 3.2. Install the packages

```bash
apt update && apt upgrade -y
apt install -y ufw fail2ban curl unattended-upgrades
```

### 3.3. Tune the kernel parameters

Make the file `/etc/sysctl.d/99-hardening.conf`.

```
# Security
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2
kernel.yama.ptrace_scope = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_syncookies = 1

# Performance
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 7500000
net.core.wmem_max = 7500000
vm.swappiness = 10
```

Make the file `/etc/sysctl.d/99-tailscale.conf`.
The exit node needs the packet forwarding.

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

Apply the two files.

```bash
sysctl --system
```

Make sure that the values are correct.

```bash
sysctl -n net.ipv4.tcp_congestion_control net.ipv4.ip_forward
```

The command must show `bbr` and `1`.

The values `rmem_max` and `wmem_max` are the values that Tailscale recommends
for an exit node.

### 3.4. Configure the firewall

WARNING: Permit the SSH port before you enable the firewall.
If you do not permit the SSH port, you lose the access to the server.

```bash
ufw default deny incoming
ufw default allow outgoing
ufw limit 22/tcp comment 'SSH rate-limited'
ufw allow 443/tcp comment 'DERP TLS'
ufw allow 443/udp comment 'peer relay'
ufw allow 3478/udp comment 'STUN'
ufw allow 41641/udp comment 'tailscale wireguard'
ufw allow in on tailscale0
ufw route allow in on tailscale0
ufw route allow out on tailscale0
ufw --force enable
```

The rule `ufw limit` rejects an address after six connections in 30 seconds.
The rejected connection shows the message `Connection refused`.
This message is misleading, because the SSH server continues to operate.
If you use an automation tool that makes many connections, use the SSH
connection multiplexing.

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%C
    ControlPersist 30m
```

### 3.5. Harden the SSH server

Make the file `/etc/ssh/sshd_config.d/00-hardening.conf`.

The prefix `00-` is necessary. The SSH server uses the first value that it
finds. The file must sort before the default files of the distribution.

```
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
AllowStreamLocalForwarding no
PermitTunnel no
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers root ubuntu
MaxStartups 3:50:10

KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-128-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
PubkeyAcceptedAlgorithms ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-256
```

CAUTION: The setting `HostKeyAlgorithms` removes the ECDSA host key.
If a client has only the ECDSA key in the file `known_hosts`, the client
shows a warning. Examine the client keys before you apply the change.

Test the configuration. Then start a rollback timer. Then restart the server.

```bash
sshd -t
cp /etc/ssh/sshd_config.d/00-hardening.conf /root/sshd.bak
systemd-run --unit=ssh-rollback --on-active=180 \
    /bin/sh -c "cp /root/sshd.bak /etc/ssh/sshd_config.d/00-hardening.conf; systemctl restart ssh"
systemctl restart ssh
```

Open a new SSH session with each permitted user.
If the new sessions operate correctly, stop the timer.

```bash
systemctl stop ssh-rollback.timer
```

If the new sessions fail, wait 180 seconds. The timer restores the previous
configuration.

### 3.6. Configure fail2ban

Make the file `/etc/fail2ban/jail.local`.

```ini
[DEFAULT]
backend = systemd
banaction = ufw
ignoreip = 127.0.0.1/8 ::1 100.64.0.0/10

[sshd]
enabled = true
maxretry = 3
findtime = 10m
bantime = 1d
```

Do not put a dynamic IP address in `ignoreip`.
The internet provider gives that address to a different person later.
That person then has an unlimited number of attempts.

Make the file `/etc/fail2ban/jail.d/recidive.conf` for the repeat offenders.

```ini
[recidive]
enabled = true
backend = auto
logpath = /var/log/fail2ban.log
banaction = ufw
bantime = 1w
findtime = 1d
maxretry = 3
```

The parameter `backend = auto` is necessary.
The jail `recidive` reads the log file of fail2ban.
It does not read the journal. The default backend in `[DEFAULT]` is
incorrect for this jail.

```bash
systemctl enable --now fail2ban
fail2ban-client status
```

### 3.7. Install the kernel live patches

A DERP server must have a high availability. The kernel updates need a
restart. The Ubuntu Pro live patch applies the kernel corrections without a
restart.

The personal subscription is free for five machines.

```bash
apt install -y ubuntu-pro-client
pro attach <YOUR_TOKEN>
pro enable livepatch
canonical-livepatch status
```

The status must show `running: true`.

## 4. Open the ports at the provider

Many providers have a firewall that is external to the server.
This firewall can block the ports, and the server cannot show this condition.

Open these ports in the control panel of the provider.

| Port | Protocol | Function |
| --- | --- | --- |
| 22 | TCP | SSH |
| 443 | TCP | DERP over TLS, and the certificate request |
| 443 | UDP | Peer relay |
| 3478 | UDP | STUN |
| 41641 | UDP | Tailscale WireGuard |

Port 80 is not necessary.

### 4.1. Test the ports from an external location

Test each port from a different machine.

```bash
nc -vz -4 203.0.113.10 443
```

If a port does not reply, find the location of the blockage.
Start a packet capture on the server. Then send the packets from the
external machine.

```bash
tcpdump -i any -n 'tcp port 443 and tcp[tcpflags] & tcp-syn != 0'
```

The `tcpdump` program operates below the firewall of the operating system.
Use this fact to identify the location of the blockage:

- If `tcpdump` shows no packets, the blockage is external to the server.
  Examine the firewall of the provider.
- If `tcpdump` shows the packets, but the connection fails, the blockage is
  on the server. Examine `ufw`.

## 5. Install Tailscale

### 5.1. Add the tag to the policy file

Add the tag owner to the tailnet policy file before you connect the server.
If the tag does not exist, the connection fails.

```json
"tagOwners": {
  "tag:relay": ["autogroup:admin"],
},
```

### 5.2. Connect the server to the tailnet

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-tags=tag:relay --hostname=derp-server
```

The command shows a URL. Open the URL in a browser to authenticate the node.

A tagged node does not have a key expiry. You do not have to disable the key
expiry manually.

Find the tailnet address of the server. You need this address later.

```bash
tailscale ip -4
```

## 6. Configure the peer relay

A peer relay moves the UDP traffic between two nodes at the network level.
It is more efficient than DERP, but it needs an open UDP port.

```bash
tailscale set --relay-server-port=443
tailscale set --accept-dns=false
tailscale set --operator=ubuntu
tailscale set --auto-update=false
```

CAUTION: Keep `--auto-update=false`. The programs `tailscaled` and `derper`
must have the same version. An automatic update changes only `tailscaled`.
Refer to section 11.

Add the capability grant to the tailnet policy file.

```json
"grants": [
  {
    "src": ["autogroup:member", "autogroup:tagged"],
    "dst": ["tag:relay"],
    "app": {"tailscale.com/cap/relay": []},
  },
],
```

Make sure that the server shows itself as a relay.

```bash
tailscale debug peer-relay-servers
```

## 7. Configure the exit node

An exit node sends the internet traffic of the other nodes through this
server.

```bash
tailscale set --advertise-exit-node
```

Approve the route in the administration console.
Select **Machines**, then the node, then **Edit route settings**.
Then enable the exit node.

### 7.1. Increase the throughput

The Linux kernel can process the UDP packets in larger groups.
This decreases the CPU load of an exit node.

```bash
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
ethtool -K "$NETDEV" rx-udp-gro-forwarding on rx-gro-list off
```

Make the file `/etc/systemd/system/tailscale-gro.service` to keep this
setting after a restart.

```ini
[Unit]
Description=Enable UDP GRO forwarding for Tailscale exit-node throughput
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now tailscale-gro
```

## 8. Build the derper program

The `derper` program is not in the Ubuntu repositories. You must build it.

### 8.1. Install Go

```bash
GOVER=$(curl -s "https://go.dev/VERSION?m=text" | head -1)
curl -fsSLo /tmp/go.tgz "https://go.dev/dl/${GOVER}.linux-amd64.tar.gz"
rm -rf /usr/local/go
tar -C /usr/local -xzf /tmp/go.tgz
export PATH=/usr/local/go/bin:$PATH
go version
```

### 8.2. Build derper

Build `derper` with the same version as `tailscaled`.

```bash
TSVER=$(tailscale version | head -1)
GOFLAGS=-buildvcs=false go install "tailscale.com/cmd/derper@v${TSVER}"
mv /root/go/bin/derper /usr/local/bin/derper
chmod 755 /usr/local/bin/derper
derper -version
```

The version can include the suffix `-ERR-BuildInfo`.
This suffix is cosmetic. It occurs when Go downloads a newer toolchain
during the build. It has no effect on the operation.

## 9. Configure the derper service

### 9.1. Make the user and the directory

```bash
useradd -r -s /usr/sbin/nologin -d /var/lib/derper derper
mkdir -p /var/lib/derper
chown derper:derper /var/lib/derper
chmod 700 /var/lib/derper
```

The directory holds the private key of the node and the TLS certificate.
Do not make the directory readable by other users.

### 9.2. Make the service unit

Make the file `/etc/systemd/system/derper.service`.

```ini
[Unit]
Description=Tailscale DERP Server
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
User=derper
Group=derper
ExecStart=/usr/local/bin/derper -c=/var/lib/derper/derper.key -hostname=derp.example.com -certmode=letsencrypt -certdir=/var/lib/derper -a=:443 -http-port=-1 -stun -stun-port=3478 -verify-clients -home=blank -accept-connection-limit=20 -accept-connection-burst=200
Restart=always
RestartSec=5
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/derper
ProtectKernelTunables=true
ProtectControlGroups=true
RestrictNamespaces=true
RestrictSUIDSGID=true
LockPersonality=true
MemoryHigh=256M
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

The flags have these functions:

| Flag | Function |
| --- | --- |
| `-c` | The path of the node key. It is necessary for a non-root user. |
| `-certmode=letsencrypt` | Get and renew the certificate automatically. |
| `-http-port=-1` | Disable the HTTP listener. |
| `-verify-clients` | Permit only the nodes of this tailnet. |
| `-home=blank` | Disable the status page. |
| `-accept-connection-limit` | Limit the rate of the new connections. |

CAUTION: The flag `-c` is necessary. The `derper` program uses a default
path only when it operates as root. This service operates as the user
`derper`. Without the flag, the program stops with an error.

The parameters `MemoryHigh` and `MemoryMax` protect the other services.
If `derper` uses too much memory, systemd stops only `derper`.
The programs `tailscaled` and `sshd` continue to operate.

NOTE: `AdGuardHome.md` gives a different rule for the DNS server: use
`MemoryHigh`, but do not use `MemoryMax`. The two rules are not contrary. If
`derper` stops, the clients use a different relay. If the DNS server stops,
the name resolution stops on all the devices of the tailnet. Use `MemoryMax`
only for a service that can stop without an effect on the other devices.

### 9.3. Start the service

```bash
systemctl daemon-reload
systemctl enable --now derper
journalctl -u derper -f
```

The program gets the certificate at the first TLS connection.
The certificate renews automatically. No other task is necessary.

## 10. Add the DERP server to the policy file

Add the DERP map to the tailnet policy file.

```json
"derpMap": {
  "OmitDefaultRegions": false,
  "Regions": {
    "900": {
      "RegionID":   900,
      "RegionCode": "custom",
      "RegionName": "Custom DERP",
      "Nodes": [
        {
          "Name":     "custom-1",
          "RegionID": 900,
          "HostName": "derp.example.com",
          "DERPPort": 443,
          "STUNPort": 3478,
          "STUNOnly": false,
        },
      ],
    },
  },
},
```

Keep `OmitDefaultRegions` as `false`.
The nodes then use the Tailscale regions if your server stops.
The nodes select the region with the lowest latency.

## 11. Keep the versions in agreement

The programs `tailscaled` and `derper` must have the same version.
A difference between the versions stops the DERP service.

Make the file `/usr/local/bin/ts-update`.

```bash
#!/bin/bash
# Update tailscaled and rebuild derper to the same version.
set -e

RED="\033[0;31m"; GREEN="\033[0;32m"; YELLOW="\033[1;33m"
CYAN="\033[0;36m"; NC="\033[0m"

if [ "$EUID" -ne 0 ]; then
    echo -e "${RED}Run this script with sudo: sudo ts-update${NC}"
    exit 1
fi

export PATH=/usr/local/go/bin:$PATH
export GOFLAGS=-buildvcs=false
GO_BIN="/root/go/bin/derper"

clean_version() {
    echo "$1" | grep -oP "^\d+\.\d+\.\d+" || true
}

BEFORE_TS=$(clean_version "$(tailscaled --version | head -1 | awk '{print $1}')")
BEFORE_DERPER=$(clean_version "$(derper -version 2>&1 | head -1 | awk '{print $1}')")
echo -e "Before: tailscaled ${YELLOW}${BEFORE_TS}${NC} | derper ${YELLOW}${BEFORE_DERPER}${NC}"

echo -e "${CYAN}[1/4] Check for the updates...${NC}"
tailscale update --yes
NEW_TS=$(clean_version "$(tailscaled --version | head -1 | awk '{print $1}')")

if [ "$NEW_TS" = "$BEFORE_TS" ] && [ "$BEFORE_TS" = "$BEFORE_DERPER" ]; then
    echo -e "${GREEN}The versions agree at ${NEW_TS}. No task is necessary.${NC}"
    exit 0
fi

echo -e "${CYAN}[2/4] Build derper v${NEW_TS}...${NC}"
go install "tailscale.com/cmd/derper@v${NEW_TS}"
[ -f "$GO_BIN" ] || { echo -e "${RED}The build failed.${NC}"; exit 1; }

echo -e "${CYAN}[3/4] Install derper...${NC}"
mv "$GO_BIN" /usr/local/bin/derper
chmod 755 /usr/local/bin/derper

echo -e "${CYAN}[4/4] Restart derper...${NC}"
systemctl restart derper
sleep 3

AFTER_TS=$(clean_version "$(tailscaled --version | head -1 | awk '{print $1}')")
AFTER_DERPER=$(clean_version "$(derper -version 2>&1 | head -1 | awk '{print $1}')")
echo -e "After: tailscaled ${GREEN}${AFTER_TS}${NC} | derper ${GREEN}${AFTER_DERPER}${NC}"

if [ "$AFTER_TS" = "$AFTER_DERPER" ]; then
    echo -e "${GREEN}The versions agree.${NC}"
else
    echo -e "${RED}The versions do not agree.${NC}"
    exit 1
fi
```

```bash
chmod 755 /usr/local/bin/ts-update
```

## 12. Show the status at the login

A message at the login shows the problems without a manual test.

Make the file `/etc/profile.d/z-custom-motd.sh`.
Refer to the repository for the full script.

The script must obey these rules:

- Do not use the command `exit`. The shell reads this file. The command
  `exit` closes the session of the user.
- Put the script in a subshell. The variables then stay local.
- Show the message only for an interactive shell. If you do not do this,
  the message damages the `scp` and `rsync` sessions.

The message shows this data:

- The versions of `tailscaled` and `derper`, and a warning if they differ.
- The condition of the services.
- The number of days before the certificate expires.

## 13. Monitor the server

The failure of the DERP server is silent. The nodes use the default regions,
and the users see no error.

Make an HTTPS monitor for `https://derp.example.com/derp/probe`.
Set the monitor to expect the code 200. Enable the SSL verification.
The monitor then also finds an expired certificate.

Make a heartbeat monitor for the other conditions.
The script sends a signal only when the server is correct.

```bash
#!/bin/bash
set -u
URL=$(cat /etc/betterstack-heartbeat.url)

fail() { echo "$1"; exit 1; }

systemctl is-active --quiet derper     || fail "derper is not active"
systemctl is-active --quiet tailscaled || fail "tailscaled is not active"
curl -fsS --max-time 20 https://derp.example.com/derp/probe -o /dev/null \
    || fail "the DERP probe failed"

curl -fsS --max-time 20 "$URL" -o /dev/null
```

Start the script with a systemd timer each hour.

## 14. Test the installation

Do these tests after the installation.

| Test | Command | Correct result |
| --- | --- | --- |
| The DERP server replies | `curl -v https://derp.example.com/derp/probe` | The code 200 |
| The certificate is valid | `openssl s_client -connect derp.example.com:443 \| openssl x509 -noout -dates` | A future date |
| The peer relay operates | `tailscale debug peer-relay-servers` | The address of the server |
| The clients see the region | `tailscale netcheck` | The custom region |
| The relay moves the traffic | `tailscale debug peer-relay-sessions` | One session or more |
| The exit node operates | Select the exit node, then `curl -4 ifconfig.me` | The address of the server |

### 14.1. Test after a restart

Restart the server one time after the installation.
Many errors occur only at the start of the operating system.

```bash
systemctl reboot
```

After the restart, make sure that these conditions are true.

```bash
systemctl --failed
swapon --show
sysctl -n net.ipv4.tcp_congestion_control
ufw status
systemctl is-active derper tailscaled
```

## 15. Troubleshooting

### 15.1. The certificate request fails

The log shows `acme/autocert: missing certificate`.

This message is not the primary error. The library keeps a failed state for
60 seconds. Each connection in that period shows this message.

Examine the full log to find the primary error.

```bash
journalctl -u derper --no-pager -o cat | grep -v "missing certificate" | tail -20
```

These are the usual primary errors.

| Error | Cause | Solution |
| --- | --- | --- |
| `unable to satisfy ...` | Let's Encrypt cannot connect to port 443. | Open the port at the provider. |
| `429 ... rateLimited` | Too many failed attempts. | Wait. The message gives the time. |
| `no such host` | The DNS record is incorrect. | Correct the record. |

CAUTION: Let's Encrypt permits five failed attempts for each host name in
one hour. Each TLS connection during a failure uses one attempt.
Do not test many times when the port is closed.

### 15.2. The DERP server does not start

Examine the version agreement first.

```bash
tailscaled --version | head -1
derper -version
```

If the versions differ, use the command `sudo ts-update`.

If the log shows `-c <config path> not specified`, add the flag `-c` to the
service unit. Refer to section 9.2.

### 15.3. The clients do not use the DERP server

Use the command `tailscale netcheck` on a client.
If the custom region is not in the list, the policy file is incorrect.
Examine the `derpMap` section.

If the region is in the list, but the latency is high, the client is far
from the server. This condition is correct.

### 15.4. The exit node does not operate

Make sure that the route is approved in the administration console.
Then make sure that the packet forwarding is enabled.

```bash
sysctl -n net.ipv4.ip_forward
```

The command must show `1`.

### 15.5. You cannot open an SSH session

The message `Connection refused` has two causes.

- The rate limit of `ufw` rejected the connection.
  Wait 30 seconds. Then try one time.
- The SSH server does not operate. Use a different access method.

If the tailnet operates, use the tailnet address.
The firewall permits the interface `tailscale0` without a rate limit.

```bash
ssh root@100.x.y.z
```

Keep the access through the public address. That method uses the SSH key.
It does not need the tailnet or the policy file.
