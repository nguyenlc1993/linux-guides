# AdGuard Home as a private DNS server on a tailnet

This guide is written in ASD-STE100 Simplified Technical English.

## 1. Purpose and scope

This guide shows how to install AdGuard Home on a server in a Tailscale
tailnet. The DNS server then gives the name resolution and the advertisement
filter to all the devices in the tailnet.

The DNS server listens only on the tailnet address.
It is not available from the internet.

This guide uses the same server as `TailscaleDERP.md`.
You can also use a different server.

### 1.1. Values in this guide

Replace these values with your own values.

| Value | Example in this guide |
| --- | --- |
| Tailnet IPv4 address of the server | `100.x.y.z` |
| Administration port | `3000` |

### 1.2. Why AdGuard Home and not Pi-hole

AdGuard Home is one program in the Go language.
The administration interface is in the program.

Pi-hole needs a web server and PHP.
It uses approximately two times more memory.

For a small server, use AdGuard Home.

## 2. Prerequisites

Before you start, make sure that these conditions are true.

- The server has Ubuntu 24.04 LTS.
- Tailscale operates on the server, and the node is authenticated.
- You have root access.
- The server has a minimum of 200 MB of free RAM.

Find the tailnet address of the server. You need this address many times.

```bash
tailscale ip -4
```

Make sure that the port 53 is free on the tailnet address.

```bash
ss -tulnp | grep ':53 '
```

The program `systemd-resolved` uses the addresses `127.0.0.53` and
`127.0.0.54`. It does not use the tailnet address.
Therefore there is no conflict.

## 3. Test the upstream resolvers

WARNING: Do this test before you configure the upstream resolvers.
Some networks block the UDP port 53 to some addresses.
A blocked resolver does not give an error. Each query waits until the
timeout. This makes the DNS service very slow.

Test each resolver that you intend to use.

```bash
for r in 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4 9.9.9.9; do
  ms=$(dig +tries=1 +time=3 @$r cloudflare.com A 2>/dev/null | awk '/Query time:/ {print $4}')
  printf "%-16s %s\n" "$r" "${ms:-UNREACHABLE}"
done
```

Use only the resolvers that give a time.

### 3.1. Identify a partial blockage

A resolver can reply to the ICMP packets but not to the DNS queries.
The command `ping` is not a sufficient test.

Test the three protocols.

```bash
ping -c 3 1.0.0.1                          # ICMP
dig +tries=1 +time=3 @1.0.0.1 example.com  # UDP port 53
dig +tcp +tries=1 +time=5 @1.0.0.1 example.com  # TCP port 53
```

If the ICMP and the TCP operate, but the UDP does not operate, the network
blocks the UDP port 53 to that address.
Section 6 shows how to prevent this problem.

## 4. Install AdGuard Home

Download and install the program.

```bash
curl -sS -L https://static.adguard.com/adguardhome/release/AdGuardHome_linux_amd64.tar.gz -o /tmp/agh.tar.gz
tar -C /opt -xzf /tmp/agh.tar.gz
rm -f /tmp/agh.tar.gz
/opt/AdGuardHome/AdGuardHome -s install
```

The program starts and listens on all the addresses at the port 3000.

### 4.1. Complete the installation

Use the installation API. Do not use the browser wizard.
The API lets you set the addresses in one step.

Make a password. Then send the configuration.

```bash
PW=$(head -c 12 /dev/urandom | base64 | tr -d '/+=' | head -c 16)
echo "The password is: $PW"

curl -sS -X POST http://127.0.0.1:3000/control/install/configure \
  -H "Content-Type: application/json" \
  -d "{\"web\":{\"ip\":\"0.0.0.0\",\"port\":3000},
       \"dns\":{\"ip\":\"100.x.y.z\",\"port\":53},
       \"username\":\"admin\",\"password\":\"$PW\"}"
```

NOTE: Set the web address to `0.0.0.0` in this step.
The installation program tests the address before it uses it.
The port 3000 is in use by the installation program itself.
The test fails if you give the final address now.
Section 5 corrects the address.

## 5. Limit the program to the tailnet

Edit the file `/opt/AdGuardHome/AdGuardHome.yaml`.
Change the web address to the tailnet address.

```yaml
http:
  address: 100.x.y.z:3000
```

Also increase the cache. The cache decreases the number of the upstream
queries. This has a large effect on the latency.

```yaml
dns:
  bind_hosts:
    - 100.x.y.z
  port: 53
  cache_size: 33554432
  cache_ttl_min: 300
  cache_optimistic: true
```

The parameter `cache_optimistic` sends the old reply immediately.
The program then gets the new reply in the background.

Restart the program. Then make sure that it listens only on the tailnet
address.

```bash
systemctl restart AdGuardHome
ss -tulnp | grep AdGuard
```

The output must show only the tailnet address.
If the output shows `0.0.0.0` or `*`, the configuration is incorrect.

## 6. Encrypt the upstream queries

DNS over TLS (DoT) sends the queries through the TCP port 853.
This has two advantages:

- The network cannot read or change the queries.
- The method prevents the blockage of the UDP port 53. Refer to section 3.1.

Edit the file `/opt/AdGuardHome/AdGuardHome.yaml`.

```yaml
dns:
  upstream_dns:
    - tls://one.one.one.one
    - tls://dns.google
    - tls://dns.quad9.net
  bootstrap_dns:
    - 1.1.1.1
    - 8.8.8.8
  upstream_mode: parallel
  upstream_timeout: 5s
```

Use the host names, and not the IP addresses.
The program then can examine the TLS certificate.
An IP address gives the encryption, but not the authentication.

The parameter `bootstrap_dns` must use the plain DNS.
It resolves the host names of the upstream servers.
Use only the addresses that operate. Refer to section 3.

The parameter `upstream_mode: parallel` sends the query to all the upstream
servers. The program uses the first reply.
The default mode is `load_balance`. That mode sends the query to one server.
If that server is unavailable, the query waits until the timeout.

Restart the program.

```bash
systemctl restart AdGuardHome
```

### 6.1. Make sure that the queries are encrypted

Start a packet capture. Then make some queries.

```bash
tcpdump -i any -n '(tcp port 853 or udp port 53) and not host 100.x.y.z'
```

The capture must show many packets at the port 853.
It must show only a small number of packets at the port 53.
Those packets are the bootstrap queries.

## 7. Start the program after the tailnet interface

WARNING: The program binds to the tailnet address.
That address does not exist at the start of the operating system.
Without this correction, the program fails and restarts many times.

The condition `After=tailscaled.service` is not sufficient.
The service `tailscaled` becomes active before the interface has an address.

Make the file `/usr/local/bin/wait-tailnet-ip`.

```bash
#!/bin/sh
# Wait for the tailnet address. Exit 0 after the timeout, because
# systemd Restart=always then continues the attempts.
for _ in $(seq 90); do
    ip -4 addr show tailscale0 2>/dev/null | grep -q "100.x.y.z" && exit 0
    sleep 1
done
exit 0
```

```bash
chmod 755 /usr/local/bin/wait-tailnet-ip
```

Make the directory and the file for the service override.

```bash
mkdir -p /etc/systemd/system/AdGuardHome.service.d
```

Make the file
`/etc/systemd/system/AdGuardHome.service.d/10-after-tailscale.conf`.

```ini
[Unit]
After=tailscaled.service
Wants=tailscaled.service

[Service]
ExecStartPre=/usr/local/bin/wait-tailnet-ip
```

```bash
systemctl daemon-reload
```

### 7.1. Test the correction

Restart the operating system. Then count the errors.

```bash
systemctl reboot
```

After the restart, use this command.

```bash
journalctl -u AdGuardHome -b --no-pager -o cat | grep -c "cannot assign requested address"
```

The result must be `0`.

## 8. Change the administration password

The version 0.107 does not have a function to change the password.
The administration interface and the API do not have this function.
The endpoint `/control/profile/update` changes only the name, the language,
and the theme.

You must edit the configuration file.

Make the file `/usr/local/bin/adguard-passwd`.

```bash
#!/bin/bash
# Change the AdGuard Home administrator password.
set -euo pipefail

CONF=/opt/AdGuardHome/AdGuardHome.yaml
STAMP=/var/lib/adguard-passwd-changed
UI=http://100.x.y.z:3000

[ "$EUID" -eq 0 ] || exec sudo -- "$0" "$@"

USERNAME=$(python3 -c "import yaml;print(yaml.safe_load(open('$CONF'))['users'][0]['name'])")

read -rsp "The new password for the user \"$USERNAME\": " P1; echo
read -rsp "Type the password again: " P2; echo
[ -n "$P1" ]      || { echo "The password is empty."; exit 1; }
[ "$P1" = "$P2" ] || { echo "The passwords are different."; exit 1; }
[ ${#P1} -ge 12 ] || { echo "Use a minimum of 12 characters."; exit 1; }

# The option -i reads the password from the standard input.
# The password then does not become visible in the process list.
HASH=$(printf "%s" "$P1" | htpasswd -niB "$USERNAME" | cut -d: -f2)

cp -a "$CONF" "$CONF.bak"
HASH="$HASH" python3 - <<'PY'
import os, yaml
p = "/opt/AdGuardHome/AdGuardHome.yaml"
c = yaml.safe_load(open(p))
c["users"][0]["password"] = os.environ["HASH"]
yaml.safe_dump(c, open(p, "w"), default_flow_style=False, sort_keys=False)
PY

systemctl restart AdGuardHome
sleep 4

CODE=$(AGH_USER="$USERNAME" AGH_PW="$P1" AGH_UI="$UI" python3 - <<'PY'
import json, os, urllib.request, urllib.error
body = json.dumps({"name": os.environ["AGH_USER"],
                   "password": os.environ["AGH_PW"]}).encode()
req = urllib.request.Request(os.environ["AGH_UI"] + "/control/login", data=body,
                             headers={"Content-Type": "application/json"})
try:
    print(urllib.request.urlopen(req, timeout=10).status)
except urllib.error.HTTPError as e:
    print(e.code)
except Exception:
    print(0)
PY
)

if [ "$CODE" = "200" ]; then
    date +%s > "$STAMP"
    rm -f "$CONF.bak"
    echo "The password is correct. Put the password in your password manager."
else
    echo "The test failed with the code $CODE. The script restores the old file."
    mv "$CONF.bak" "$CONF"
    systemctl restart AdGuardHome
    exit 1
fi
```

```bash
apt install -y apache2-utils
chmod 755 /usr/local/bin/adguard-passwd
```

The script tests the new password before it keeps the change.
If the test fails, the script restores the old file.
You cannot lose the access.

CAUTION: Use the Python library to edit the file. Do not use `sed`.
A bcrypt hash contains the characters `$` and `/`.
The program `sed` gives an incorrect result with these characters.

## 9. Set the DNS server for the tailnet

Open the administration console of Tailscale.
Select **DNS**. Then add the tailnet address of the server as a name server.
Then enable **Override local DNS**.

CAUTION: Use only one name server. Tailscale sends the queries to all the
name servers in the list. A second name server therefore decreases the
quantity of the filtered queries.

CAUTION: All the devices then depend on this server for the DNS.
If the server stops, the name resolution stops on all the devices.
To correct this condition, disable **Override local DNS**.

## 10. Test the installation

### 10.1. Test on the server

```bash
dig +short @100.x.y.z github.com A          # must give an address
dig +short @100.x.y.z doubleclick.net A     # must give 0.0.0.0
```

### 10.2. Test on a client

Tailscale does not put the address of your server in the configuration of
the operating system. It uses its own resolver at the address
`100.100.100.100`. That resolver sends the queries to your server.
This condition is correct.

On macOS, use this command to examine the configuration.

```bash
tailscale dns status
```

The output must show your server in the list `Resolvers`.

Test the full path of the operating system, and not only the command `dig`.

```bash
dscacheutil -q host -a name doubleclick.net    # macOS
getent hosts doubleclick.net                   # Linux
```

The result must be `0.0.0.0`.

### 10.3. Measure the latency

```bash
dig @100.100.100.100 google.com A | grep "Query time"
```

Make the same query two times. The second query uses the cache.

These are the typical results for a server at a distance of 30 ms.

| Type of the query | Latency |
| --- | --- |
| A blocked name | The latency to the server |
| A name in the cache | The latency to the server |
| A new name | The latency to the server, plus the latency to the upstream |

## 11. Monitor the DNS server

The DNS server is only on the tailnet. An external monitor cannot test it.
Use a script on the server, and a heartbeat monitor.

The script must test three conditions:

- The service is active.
- The name resolution operates.
- The filter operates.

```bash
TSIP=100.x.y.z

systemctl is-active --quiet AdGuardHome || fail "AdGuardHome is not active"

dig +short +time=5 +tries=2 @"$TSIP" github.com A | grep -qE '^[0-9]+\.' \
    || fail "the name resolution failed"

[ "$(dig +short +time=5 +tries=2 @"$TSIP" doubleclick.net A | head -1)" = "0.0.0.0" ] \
    || fail "the filter does not operate"
```

The third test is necessary. The program can resolve the names correctly
but not apply the filter. The first two tests do not find this condition.

## 12. Troubleshooting

### 12.1. The queries are very slow

Examine the log for the upstream errors.

```bash
journalctl -u AdGuardHome --no-pager -o cat | grep -i "exchange failed" | tail -5
```

An error with `i/o timeout` shows a blocked or unavailable upstream server.
Remove that server from the configuration. Refer to section 3.
Or use DoT. Refer to section 6.

### 12.2. The program does not start after a restart

Examine the log for the errors of the address.

```bash
journalctl -u AdGuardHome -b --no-pager -o cat | grep -i "cannot assign"
```

If the log shows the errors, the program started before the tailnet
interface. Refer to section 7.

### 12.3. The clients do not use the filter

Make sure that **Override local DNS** is enabled in the administration
console.

Then make sure that there is only one name server in the list.
If there are two name servers, some queries do not use your server.

### 12.4. You cannot open the administration interface

The interface listens only on the tailnet address.
Connect the client to the tailnet first.

Then open `http://100.x.y.z:3000`.

If you do not have the password, refer to section 8.
