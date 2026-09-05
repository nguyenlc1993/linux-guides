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

## 10. Select the blocklists

AdGuard Home installs with one blocklist. Add more lists to increase the
quantity of the blocked domains.

In the administration interface, select **Filters**, then **DNS blocklists**,
then **Add blocklist**, then **Choose from the list**. The catalog comes from
this file:

```
https://adguardteam.github.io/HostlistsRegistry/assets/filters.json
```

### 10.1. A sufficient set

This set gives good coverage for English and Vietnamese web sites.

| List | Rules | Function |
| --- | --- | --- |
| HaGeZi's Threat Intelligence Feeds - Medium | 321 100 | Malware, phishing and command-and-control |
| HaGeZi's Pro++ Blocklist | 250 500 | General advertisements, trackers and telemetry |
| AdGuard DNS filter | 180 000 | General advertisements and trackers |
| HaGeZi NSFW Blocklist | 125 600 | Adult content |
| Phishing URL Blocklist | 37 700 | Phishing |
| VNM: ABPVN List | 19 100 | Vietnamese web sites |
| AdAway Default Blocklist | 6 500 | Advertisements in mobile applications |
| Malicious URL Blocklist (URLHaus) | 4 800 | Malware |
| Peter Lowe's Blocklist | 3 500 | Advertisements and trackers |
| HaGeZi's DynDNS Blocklist | 1 500 | Dynamic DNS that malware uses |
| AdGuard DNS Popup Hosts filter | 1 000 | Pop-up windows |

The total is 952 000 rules. Of these rules, 799 000 are unique. The lists thus
have an overlap of 16 per cent. This overlap is not a defect. Different lists
find different domains.

CAUTION: Before you add a list, test it against the names that your own
system needs. Examine the parent domains, and not only the full names. A list
can block `dns.quad9.net` with the rule `||quad9.net^` for the parent domain,
and a search for the full name does not find that rule.

```bash
python3 - <<'PY'
rules = {l.strip() for l in open("/opt/AdGuardHome/data/filters/<ID>.txt")}
for d in ["one.one.one.one", "dns.google", "dns.quad9.net",
          "controlplane.tailscale.com", "derp.example.com"]:
    p = d.split(".")
    hit = next((".".join(p[i:]) for i in range(len(p) - 1)
                if "||%s^" % ".".join(p[i:]) in rules), None)
    print(d, "->", hit or "not blocked")
PY
```

These small lists have a low cost and a good result.

| List | Rules | Function |
| --- | --- | --- |
| HaGeZi's Badware Hoster Blocklist | 1 200 | Hosts that supply malware |
| HaGeZi's Apple Tracker Blocklist | 107 | Telemetry of macOS and iOS |
| HaGeZi's DNS Rebind Protection | 16 | DNS rebinding attacks |

CAUTION: The list `DNS Rebind Protection` blocks all the answers that contain
a private address. Development tools such as `nip.io`, `sslip.io` and `lvh.me`
give a private address. These tools stop if you add this list. The list does
not block the range `100.64.0.0/10`. The tailnet thus continues to operate.

NOTE: To remove a list, set it to disabled. Do not delete it. A disabled list
uses no memory, because the program does not read it. But you can enable it
again immediately.

### 10.2. Lists to prevent

| List | Rules | Reason |
| --- | --- | --- |
| HaGeZi's Threat Intelligence Feeds | 2 170 000 | Too large for a server with 1 GB of memory |
| HaGeZi's Gambling Blocklist | 461 000 | It increases the total by 90 per cent |
| HaGeZi's URL Shortener Blocklist | 9 900 | It blocks `bit.ly` and other necessary services |
| HaGeZi's Encrypted DNS/VPN/TOR/Proxy Bypass | 16 500 | It blocks Tailscale. Refer to the warning below |

WARNING: Do not use `Encrypted DNS/VPN/TOR/Proxy Bypass` on a tailnet. The
function of that list is to stop the devices that go around your DNS server.
It therefore blocks the VPN, the proxy and the tunnel services. Tailscale is a
VPN. The list contains `||tailscale.com^`, which blocks
`controlplane.tailscale.com` — the coordination server of your own tailnet.

The failure is not immediate, and it is thus difficult to find. The tailnet
continues to operate with the connections that exist. But the devices cannot
get a new authentication, and you cannot open the administration console.

The list has 16 494 rules for the domains. Of these rules, 10 273 block a full
domain. It also blocks these services:

| Name | Function |
| --- | --- |
| `mask.icloud.com`, `mask-h2.icloud.com` | iCloud Private Relay |
| `ngrok.com`, `ngrok.io` | Tunnels for the development |
| `nextdns.io`, `mullvad.net` | Other DNS and VPN services |
| `mozilla.cloudflare-dns.com` | DNS over HTTPS of Firefox |
| `cloudflare-gateway.com` | Cloudflare Zero Trust |

The list gives one advantage: it stops the DNS over HTTPS of the browsers,
which goes around your server. But a tailnet is a VPN, and this list stops the
VPN services. The two functions are contrary. Do not use the list.

Do not add a list for a device that you do not have. The lists for Samsung,
Xiaomi, Windows, OPPO and Vivo have no function on a tailnet of Apple devices.

NOTE: `Threat Intelligence Feeds` has smaller versions. The catalog of AdGuard
Home does not contain them. Add the version `Medium` or `Mini` with these
addresses.

```
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.medium.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.mini.txt
```

A larger list does not always give more duplicates. These are the measurements
against the other lists in section 10.1.

| Version | Rules | Already covered | New |
| --- | --- | --- | --- |
| Mini | 173 600 | 26.8 per cent | 127 100 |
| Medium | 321 100 | 19.7 per cent | 258 000 |

The version `Medium` has fewer duplicates, because its additional rules are
threat data. The lists for advertisements do not contain that data.

NOTE: `Mini` is not a subset of `Medium`. 588 rules of `Mini` are not in
`Medium`. The two lists come from different times of the build.

### 10.3. The memory budget

AdGuard Home keeps all the rules in the memory. A server with 1 GB of memory
therefore has a limit.

These are the measurements on a server with 961 MB of memory.

| Condition | Rules | Memory |
| --- | --- | --- |
| Usual operation | 952 000 | 147 MB |
| Usual operation | 968 000 | 205 MB |
| Usual operation | 843 000 | 140 MB to 195 MB |
| A restart, with the lists on the disk | 603 000 | 215 MB |
| A restart, with the lists on the disk | 843 000 | 259 MB |
| A restart, with the lists on the disk | 952 000 | 285 MB |
| A download of 4 new lists | 695 000 | 320 MB |
| A download of 1 new list | 843 000 | 431 MB |
| A download of 1 new list | 968 000 | 523 MB |
| The first download of all the 9 lists | 603 000 | 519 MB |

The download of the lists costs more memory than the restart. During a
download, the program holds the data of the download and the rules at the same
time. The program downloads the lists again at the interval
`filters_update_interval`. The default interval is 24 hours.

CAUTION: The quantity of the rules does not give the memory of a download. The
quantity of the lists that the program downloads at the same time gives the
memory. A download of 9 lists used 519 MB. A download of 4 larger lists used
only 320 MB. Measure the peak after you make a change. Do not calculate it.

A server with 1 GB of memory operates correctly with 968 000 rules. The usual
operation then uses 205 MB, and the largest download used 523 MB. That is
below the limit of 750 MB in section 10.4. Do not go above approximately
1 000 000 rules without a new measurement.

NOTE: The set in section 10.1 gives 952 000 rules and uses 147 MB. The value
of 968 000 rules includes the list `Encrypted DNS/VPN/TOR/Proxy Bypass`, which
section 10.2 tells you not to use.

### 10.4. Limit the memory of the service

Use a soft limit.

```bash
mkdir -p /etc/systemd/system/AdGuardHome.service.d
cat > /etc/systemd/system/AdGuardHome.service.d/20-memory.conf <<'EOF'
[Service]
# Soft brake on the filter-download spike (~520M peak observed).
# No MemoryMax on purpose: an OOM kill here takes tailnet DNS down.
MemoryHigh=750M
EOF
systemctl daemon-reload
```

CAUTION: Do not set `MemoryMax`. The parameter `MemoryHigh` decreases the
speed of the process and recovers the memory. The parameter `MemoryMax` stops
the process. If this process stops, the name resolution stops on all the
devices of the tailnet.

The command `systemctl daemon-reload` applies the limit to the process that
operates. A restart is not necessary. Make sure that the limit is active.

```bash
cat /sys/fs/cgroup/system.slice/AdGuardHome.service/memory.high
```

### 10.5. Measure the rules and the memory

The file `AdGuardHome.yaml` shows the value `rules_count` as 0 until the
program writes the configuration. Count the rules on the disk.

CAUTION: Do not count all the files in the directory `filters`. That directory
also contains the lists that are disabled, and the count is then too large.
Read the configuration, and count only the lists that are enabled.

```bash
python3 - <<'PY'
import os, yaml
c = yaml.safe_load(open("/opt/AdGuardHome/AdGuardHome.yaml"))
rules = set()
for f in c["filters"]:
    if not f["enabled"]:
        continue
    p = "/opt/AdGuardHome/data/filters/%d.txt" % f["id"]
    if not os.path.exists(p):
        continue
    for line in open(p, encoding="utf-8", errors="replace"):
        line = line.strip()
        if line and not line.startswith(("!", "#")):
            rules.add(line)
print("unique enabled rules:", len(rules))
PY
```

Examine the memory of the service.

```bash
systemctl show AdGuardHome -p MemoryCurrent -p MemoryPeak -p MemoryHigh
cat /sys/fs/cgroup/system.slice/AdGuardHome.service/memory.events
```

In the file `memory.events`, the counter `high` must be 0. A value that is more
than 0 shows that the process touched the limit.

NOTE: A kernel before version 6.11 cannot set `memory.peak` to 0. To get a new
value, restart the service. The restart makes a new cgroup.

## 11. Configure the protections

### 11.1. SafeSearch

SafeSearch operates. It has no dependence on the network. The program does not
send a request to a service. It only changes the answer of the DNS.

In **Settings**, then **General settings**, enable **Use SafeSearch**. Or use
this configuration.

```yaml
filtering:
  safe_search:
    enabled: true
    bing: true
    duckduckgo: true
    ecosia: true
    google: true
    pixabay: true
    yandex: true
    youtube: true
```

The rules are in the program, at
`internal/filtering/safesearch/rules/`. In the version 0.107.79, the file
`google.txt` contains 194 rules, and all the 194 rules have the prefix `|www.`.
The file `youtube.txt` contains 5 rules: `www.youtube.com`, `m.youtube.com`,
`youtubei.googleapis.com`, `youtube.googleapis.com` and
`www.youtube-nocookie.com`.

```
|www.google.com.vn^$dnsrewrite=NOERROR;CNAME;forcesafesearch.google.com
```

CAUTION: The prefix `|` finds only the full name. It does not find the
sub-domains. The rules therefore do not contain the primary domains
`google.com`, `youtube.com` or `google.com.vn`. A user that types
`google.com` does not get SafeSearch. But the rules do contain
`www.google.com.vn`. Test the primary domain and the name with `www.`
separately.

Add these rewrites in **Filters**, then **DNS rewrites**.

| Domain | Answer |
| --- | --- |
| `google.com` | `forcesafesearch.google.com` |
| `google.com.vn` | `forcesafesearch.google.com` |
| `youtube.com` | `restrictmoderate.youtube.com` |

CAUTION: If you write a rewrite in the file `AdGuardHome.yaml`, add
`enabled: true`. The program makes the value `false` if the value is not
there. The rewrite is then in the file, but it has no result.

```yaml
filtering:
  rewrites:
    - domain: google.com
      answer: forcesafesearch.google.com
      enabled: true
```

Test the rewrites. The sub-domains of Google must not change.

```bash
dig +short @100.x.y.z google.com          # forcesafesearch.google.com
dig +short @100.x.y.z youtube.com         # restrictmoderate.youtube.com
dig +short @100.x.y.z mail.google.com     # a usual address
dig +short @100.x.y.z googleapis.com      # a usual address
```

### 11.2. Give a second path for the queries

The upstream servers use DoT on the port 853. If that port stops, the name
resolution stops on all the devices. The parameter `fallback_dns` gives a
second path. The program uses it only if all the upstream servers fail.

```yaml
dns:
  fallback_dns:
    - 1.1.1.1
    - 8.8.8.8
```

Use plain addresses, and not DoT. The two paths are then different. A failure
of DoT does not stop the second path.

### 11.3. Configure the query log

The query log contains each name that each device requested. The default time
is 90 days.

```yaml
querylog:
  interval: 30d
dns:
  anonymize_client_ip: false
```

The parameter `anonymize_client_ip` removes the last part of the address of
the client in the log and in the statistics.

On a tailnet with one user, keep this parameter as `false`. The log is on your
own server, the file is readable only by root, and the clients are your own
devices. The parameter therefore hides your devices from you. It also removes
the only method to see which device made a query.

Make it `true` only if a different person can read the log.

NOTE: The parameter changes only the data in the log. The program continues to
use the correct address for the rules of each client.

NOTE: The program keeps the new entries of the log in the memory for some
minutes before it writes them to the disk. After a change, the file does not
show the result immediately.

### 11.4. Safe Browsing and Parental Control do not operate here

Do not enable these two functions if your provider examines the network.

The functions do not use a local list. For each query, the program makes a
request to `https://family.adguard-dns.com/dns-query`. That address is in the
program. You cannot change it.

The failure is not a closed port. The ports 443, 80 and 853 are open, and ICMP
gives an answer in 37 ms. But the provider examines the name in the TLS
ClientHello.

| Test | Result |
| --- | --- |
| TCP to `94.140.14.15:443` | Open |
| TLS with the name `family.adguard-dns.com` | No answer |
| TLS with no name, same address | Correct. The certificate is valid |
| The same test with IPv6 | No answer |
| TLS to `cloudflare-dns.com` | HTTP 200 in 0.16 s |

Use this test before you enable the two functions.

```bash
timeout 10 openssl s_client -connect 94.140.14.15:443 \
  -servername family.adguard-dns.com </dev/null 2>&1 | grep -c "Verify return"
```

A result of 0 shows that you cannot use these functions.

The blocklists in section 10.1 give the same result. They operate from the
disk. They have no dependence on the network, and they add no time to a query.

| Function | The equivalent list |
| --- | --- |
| Safe Browsing | `TIF Medium`, `PhishTank and OpenPhish`, `URLHaus` |
| Parental Control | `HaGeZi NSFW Blocklist` |

## 12. Test the installation

### 12.1. Test on the server

```bash
dig +short @100.x.y.z github.com A          # must give an address
dig +short @100.x.y.z doubleclick.net A     # must give 0.0.0.0
```

### 12.2. Test on a client

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

### 12.3. Measure the latency

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

## 13. Monitor the DNS server

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

## 14. Troubleshooting

### 14.1. The queries are very slow

Examine the log for the upstream errors.

```bash
journalctl -u AdGuardHome --no-pager -o cat | grep -i "exchange failed" | tail -5
```

An error with `i/o timeout` shows a blocked or unavailable upstream server.
Remove that server from the configuration. Refer to section 3.
Or use DoT. Refer to section 6.

### 14.2. The program does not start after a restart

Examine the log for the errors of the address.

```bash
journalctl -u AdGuardHome -b --no-pager -o cat | grep -i "cannot assign"
```

If the log shows the errors, the program started before the tailnet
interface. Refer to section 7.

### 14.3. The clients do not use the filter

Make sure that **Override local DNS** is enabled in the administration
console.

Then make sure that there is only one name server in the list.
If there are two name servers, some queries do not use your server.

### 14.4. You cannot open the administration interface

The interface listens only on the tailnet address.
Connect the client to the tailnet first.

Then open `http://100.x.y.z:3000`.

If you do not have the password, refer to section 8.

### 14.5. You cannot open the Tailscale administration console

A blocklist can block `tailscale.com`. Test it.

```bash
dig +short @100.x.y.z login.tailscale.com A
```

A result of `0.0.0.0` shows that a list blocks the domain. Find the list.

```bash
grep -l "||tailscale.com^" /opt/AdGuardHome/data/filters/*.txt
```

Disable that list, or add these rules in **Custom filtering rules**.

```
@@||tailscale.com^
@@||tailscale.io^
@@||tailscale.net^
```

Refer to section 10.2. The same rule also blocks
`controlplane.tailscale.com`, and the devices then cannot get a new
authentication.
