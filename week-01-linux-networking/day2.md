# Week 1, Day 2: Services, Networking & Troubleshooting

## Overview
**Goal:** Understand how services are managed on Linux, and master the networking layers + tools that let you debug real production issues.

**What you'll learn:** How to keep backend services alive, understand the network stack, and use professional debugging tools.

---

## Part 1: systemd — The Service Manager

### Mental Model

**systemd** is the **boss of all services** on Linux. Think of it like a manager:
- **Starts** services at boot
- **Keeps them running** (restarts if they crash)
- **Logs everything** they do
- **Lets you control** them (start/stop/restart)

Without systemd, a crashed database would stay dead. With it, systemd restarts it automatically.

---

### View Running Services

**Command:**
```bash
systemctl list-units --type=service --state=running
```

**Output:**
```
  UNIT                        LOAD   ACTIVE SUB     DESCRIPTION
  chrony.service              loaded active running chrony, an NTP client/server
  cron.service                loaded active running Regular background program processing daemon
  dbus.service                loaded active running D-Bus System Message Bus
  getty@tty1.service          loaded active running Getty on tty1
  networkd-dispatcher.service loaded active running Dispatcher daemon for systemd-networkd
  rsyslog.service             loaded active running System Logging Service
  systemd-journald.service    loaded active running Journal Service
  systemd-logind.service      loaded active running User Login Management
  systemd-resolved.service    loaded active running Network Name Resolution
  systemd-udevd.service       loaded active running Rule-based Manager for Device Events and Files
  unattended-upgrades.service loaded active running Unattended Upgrades Shutdown
  user@1000.service           loaded active running User Manager for UID 1000
  wsl-pro.service             loaded active running Bridge to Ubuntu Pro agent on Windows

Legend: LOAD   → Unit definition properly loaded.
        ACTIVE → High-level activation state.
        SUB    → Low-level activation state.

13 loaded units listed.
```

**What you see:**
- **13 services running** — each is `ACTIVE running`
- **chrony** = NTP time sync
- **cron** = scheduled tasks
- **rsyslog** = log collection
- **systemd-journald** = system journal
- **systemd-resolved** = DNS resolution

---

## Part 2: Control Services with systemctl

### The Pattern

```bash
systemctl <COMMAND> <service-name>
```

### Commands

| Command | What it does |
|---------|-------------|
| `status` | Check if running, when started, resource usage |
| `start` | Turn the service on |
| `stop` | Turn the service off |
| `restart` | Stop then start immediately |
| `enable` | Start at boot (persistent) |
| `disable` | Don't start at boot |

---

### Check Service Status

**Command:**
```bash
systemctl status cron.service
```

**Output:**
```
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-08-24 19:10:49 EAT; 10min ago
 Invocation: e4ab4bec6a16441abe184f4b5843425c
       Docs: man:cron(8)
   Main PID: 125 (cron)
      Tasks: 1 (limit: 3386)
     Memory: 864K (peak: 2.4M)
        CPU: 77ms
     CGroup: /system.slice/cron.service
             └─125 /usr/sbin/cron -f -P

Aug 24 19:10:49 DESKTOP-IMV74CA systemd[1]: Started cron.service - Regular background program processing daemon.
Aug 24 19:10:49 DESKTOP-IMV74CA (cron)[125]: cron.service: Referenced but unset environment variable evaluates to an empty >
Aug 24 19:10:49 DESKTOP-IMV74CA cron[125]: (CRON) INFO (pidfile fd = 3)
Aug 24 19:10:49 DESKTOP-IMV74CA cron[125]: (CRON) INFO (Running @reboot jobs)
Aug 24 19:17:01 DESKTOP-IMV74CA CRON[561]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
Aug 24 19:17:01 DESKTOP-IMV74CA CRON[563]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
Aug 24 19:17:01 DESKTOP-IMV74CA CRON[561]: pam_unix(cron:session): session closed for user root
```

**Breaking it down:**

| Field | Meaning |
|-------|---------|
| `● cron.service` | Service name (● = healthy) |
| `Loaded: loaded (...; enabled; ...)` | systemd knows about it, enabled at boot |
| `Active: active (running)` | Currently running |
| `since Mon 2026-08-24 19:10:49 EAT; 10min ago` | Started 10 minutes ago |
| `Main PID: 125 (cron)` | Process ID is 125 |
| `Memory: 864K (peak: 2.4M)` | Using 864KB, peak was 2.4MB |
| `CPU: 77ms` | Total CPU time used |

**The logs below show:**
- Service started at 19:10:49
- Hourly cron job ran at 19:17:01
- Jobs execute as scheduled

---

### View Service Logs

**Command:**
```bash
journalctl -u cron.service
```

**Output:**
```
Aug 22 13:35:43 DESKTOP-IMV74CA systemd[1]: Started cron.service - Regular background program processing daemon.
Aug 22 13:35:43 DESKTOP-IMV74CA (cron)[158]: cron.service: Referenced but unset environment variable evaluates to an empty >
Aug 22 13:35:43 DESKTOP-IMV74CA cron[158]: (CRON) INFO (pidfile fd = 3)
Aug 22 13:35:43 DESKTOP-IMV74CA cron[158]: (CRON) INFO (Running @reboot jobs)
Aug 22 14:17:01 DESKTOP-IMV74CA CRON[1068]: pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
Aug 22 14:17:01 DESKTOP-IMV74CA CRON[1070]: (root) CMD (cd / && run-parts --report /etc/cron.hourly)
Aug 22 14:17:01 DESKTOP-IMV74CA CRON[1068]: pam_unix(cron:session): session closed for user root
Aug 22 15:05:50 DESKTOP-IMV74CA systemd[1]: Stopping cron.service - Regular background program processing daemon...
Aug 22 15:05:50 DESKTOP-IMV74CA systemd[1]: cron.service: Deactivated successfully.
Aug 22 15:05:50 DESKTOP-IMV74CA systemd[1]: Stopped cron.service - Regular background program processing daemon.
```

**What you see:**
- **13:35:43** — Service started
- **14:17:01** — Hourly job executed
- **15:05:50** — Service stopped

**This is production debugging gold.** When a service crashes, you check here to see **why and when**.

---

### Restart a Service

**Command:**
```bash
sudo systemctl restart cron.service
```

**Then check status:**
```bash
systemctl status cron.service
```

**Output (after restart):**
```
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-08-24 19:26:09 EAT; 1s ago
 Invocation: 316e1d6e7c2241528c309b3d19765ac1
   Main PID: 690 (cron)
      Tasks: 1 (limit: 3386)
     Memory: 456K (peak: 1.7M)
        CPU: 21ms
```

**Notice:**
- **PID changed from 125 to 690** (new process)
- **Started 1s ago** (fresh restart)
- **Memory reset** (new process state)

This is how you verify a restart worked.

---

## Part 3: Understanding the Network Stack (OSI Model)

### The 7 Layers

```
Layer 7: APPLICATION      (HTTP, HTTPS, DNS, FTP, SSH)
Layer 6: PRESENTATION     (Encryption, compression)
Layer 5: SESSION          (Connection management)
Layer 4: TRANSPORT        (TCP, UDP — how data flows)
Layer 3: NETWORK          (IP addressing, routing)
Layer 2: LINK             (MAC addresses, switches)
Layer 1: PHYSICAL         (Cables, radio waves)
```

### What You Care About (Layers 3-7)

**Layer 3 — Network (IP)**
- IP addresses: `192.168.1.1`, `8.8.8.8`, `2001:db8::1`
- Routes data between networks
- "Where is this computer on the internet?"

**Layer 4 — Transport (TCP vs UDP)**

| Protocol | Reliable? | Connection? | Speed | Use Case |
|----------|-----------|------------|-------|----------|
| **TCP** | Yes | Yes (3-way handshake) | Slower | HTTP, databases, SSH — anything that needs every packet |
| **UDP** | No | No (just send) | Faster | DNS, video, metrics — can lose packets |

**TCP 3-Way Handshake:**
```
Client → Server: SYN (synchronize)
Server → Client: SYN-ACK (acknowledged)
Client → Server: ACK (connection established)
```

Now data flows reliably.

**UDP** skips all this — just sends packets. If one gets lost, too bad. But it's **10x faster** — perfect for DNS queries where one loss doesn't matter.

**Layer 7 — Application (HTTP/HTTPS)**
- Your backend lives here
- HTTP = unencrypted
- HTTPS = encrypted with TLS

---

## Part 4: DNS (Domain Name System)

### What DNS Does

Convert **human-readable names** to **IP addresses**.

```
google.com (name) ──DNS Query──> 142.251.36.206 (IP)
```

### DNS Query in Action

**Command:**
```bash
dig google.com
```

**Output:**
```
; <<>> DiG 9.20.18-1ubuntu2-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46092
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             46      IN      A       142.251.36.206

;; Query time: 156 msec
;; SERVER: 10.255.255.254#53(10.255.255.254) (UDP)
;; WHEN: Mon Aug 24 19:40:26 EAT 2026
;; MSG SIZE  rcvd: 55
```

**Breaking it down:**

| Part | Meaning |
|------|---------|
| `QUESTION SECTION: google.com. IN A` | "Give me the A record (IPv4) for google.com" |
| `ANSWER SECTION: google.com. 46 IN A 142.251.36.206` | **Answer: 142.251.36.206** |
| `46` | **TTL = 46 seconds** (cache this answer for 46 seconds, then re-query) |
| `Query time: 156 msec` | DNS lookup took 156 milliseconds |
| `SERVER: 10.255.255.254#53 (UDP)` | Queried DNS server on port 53 using UDP |

### DNS Records

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | IPv4 address | `google.com → 142.251.36.206` |
| **AAAA** | IPv6 address | `google.com → 2607:f8b0:4004:80b::200e` |
| **CNAME** | Alias to another domain | `www.google.com → google.com` |
| **MX** | Mail server | `gmail.com mail server` |
| **NS** | Name server | Which server handles this domain |

### TTL (Time To Live)

```
google.com.  46  IN  A  142.251.36.206
             ^^
             TTL = 46 seconds
```

**Meaning:** "Cache this result for 46 seconds. After that, ask the DNS server again."

**Why?** Reduces queries, but means IP changes take time to propagate.

---

## Part 5: TLS/HTTPS (Encryption & Certificates)

### What TLS Does

1. **Encrypts** the connection (so hackers can't see your data)
2. **Proves server identity** (so you know you're talking to real google.com, not a fake)
3. **Uses certificates** that **expire** (huge source of outages!)

### TLS Handshake in Action

**Command:**
```bash
curl -v https://google.com 2>&1 | head -30
```

**Output:**
```
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total  Spent   Left   Speed
  0      0   0      0   0      0      0      0      0:00:00 00:02   0:00:00     0* Host google.com:443 was resolved.
* IPv6: 2a00:1450:4006:818::200e
* IPv4: 142.251.142.78
*   Trying [2a00:1450:4006:818::200e]:443...
* Immediate connect fail for 2a00:1450:4006:818::200e: Network is unreachable
*   Trying 142.251.142.78:443...
* ALPN: curl offers h2,http/1.1
} [5 bytes data]
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
} [1565 bytes data]
* SSL Trust Anchors:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
*   CApath: /etc/ssl/certs
  0      0   0      0   0      0      0      0      0:00:03 00:02   0:00:01     0{ [5 bytes data]
* TLSv1.3 (IN), TLS handshake, Server hello (2):
{ [1210 bytes data]
* TLSv1.3 (IN), TLS handshake, Change cipher spec (1):
{ [1 bytes data]
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
{ [15 bytes data]
* TLSv1.3 (IN), TLS handshake, Certificate (11):
{ [4832 bytes data]
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
{ [80 bytes data]
* TLSv1.3 (IN), TLS handshake, Finished (20):
{ [52 bytes data]
* TLSv1.3 (OUT), TLS change cipher spec (1):
{ [1 bytes data]
```

**Breaking it down:**

| Step | What happens |
|------|--------------|
| `Host google.com:443 was resolved` | DNS resolved to 142.251.142.78 |
| `Trying 142.251.142.78:443` | TCP connecting to port 443 (HTTPS port) |
| `Client hello (1)` | Client sends: "I want to talk securely, here's my capabilities" |
| `Server hello (2)` | Server sends: "OK, here's my TLS version and cipher suite" |
| `Certificate (11)` | Server sends: "Here's my certificate proving I'm google.com" |
| `CERT verify (15)` | Server proves it owns the certificate |
| `Finished (20)` | Handshake complete, connection encrypted |

**Result:** Data is now encrypted. No hacker can see your traffic.

### Certificate Expiration (The Killer)

Certificates **expire**. If you forget to renew:

```
curl https://expired-cert.com
curl: (60) SSL: certificate problem: certificate has expired
```

**This brings down your entire site.** Senior engineers get paged at 3am for this.

---

## Part 6: Networking Tools for Debugging

### 1. `dig` — DNS Queries

**Check if DNS is working:**
```bash
dig google.com
```

**Check a specific record type:**
```bash
dig google.com MX
```

**Reverse lookup (IP to name):**
```bash
dig -x 142.251.36.206
```

---

### 2. `ss` — Socket Statistics (What's Listening?)

**See all listening services:**
```bash
ss -tuln
```

**Output:**
```
Netid State  Recv-Q Send-Q  Local Address:Port Peer Address:Port
udp   UNCONN 0      0          127.0.0.54:53        0.0.0.0:*
udp   UNCONN 0      0       127.0.0.53%lo:53        0.0.0.0:*
udp   UNCONN 0      0      10.255.255.254:53        0.0.0.0:*
udp   UNCONN 0      0           127.0.0.1:323       0.0.0.0:*
tcp   LISTEN 0      4096       127.0.0.54:53        0.0.0.0:*
tcp   LISTEN 0      1000   10.255.255.254:53        0.0.0.0:*
tcp   LISTEN 0      4096    127.0.0.53%lo:53        0.0.0.0:*
```

**Breaking it down:**

| Column | Meaning |
|--------|---------|
| `Netid` | Protocol (tcp, udp) |
| `State` | LISTEN (waiting for connections), UNCONN (UDP, stateless) |
| `Local Address:Port` | What this service is listening on |
| `:53` | **Port 53 = DNS** |
| `:323` | **Port 323 = NTP (time sync)** |

**Real-world example:** If your backend should listen on port 3000 but you don't see it here, it's not running!

**Flags:**
- `-t` = TCP
- `-u` = UDP
- `-l` = LISTEN only
- `-n` = show numbers (don't resolve names)

---

### 3. `curl` — Make HTTP Requests & Inspect

**Simple request:**
```bash
curl https://google.com
```

**With headers:**
```bash
curl -i https://google.com
```

**Verbose (see everything):**
```bash
curl -v https://google.com
```

**Check response status:**
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://google.com
```

This outputs just the HTTP status code (200, 404, 500, etc.).

---

## Part 7: Creating a Real Service (Practical Example)

### Create a Simple App Script

**Command:**
```bash
mkdir -p ~/my-app
cat > ~/my-app/app.sh << 'EOF'
#!/bin/bash
echo "App started at $(date)"
while true; do
  echo "$(date): App running..."
  sleep 5
done
EOF

chmod +x ~/my-app/app.sh
```

**Make it executable:**
```bash
chmod +x ~/my-app/app.sh
```

---

### Run It in Background

**Command:**
```bash
~/my-app/app.sh &
sleep 2
ps aux | grep app.sh
```

**Output:**
```
[1] 1864
App started at Mon Aug 24 20:46:35 EAT 2026
Mon Aug 24 20:46:35 EAT 2026: App running...
umer_mu+    1864  0.0  0.1   4948  3560 pts/2    S    20:46   0:00 /bin/bash /home/umer_muhammed/my-app/app.sh
umer_mu+    1880  0.0  0.0   4260  2340 pts/2    S+   20:46   0:00 grep --color=auto app.sh
Mon Aug 24 20:46:40 EAT 2026: App running...
Mon Aug 24 20:46:45 EAT 2026: App running...
Mon Aug 24 20:46:50 EAT 2026: App running...
```

**What you see:**
- **[1] 1864** = Background job, PID 1864
- **App started at Mon Aug 24 20:46:35**
- **Outputs every 5 seconds** as expected
- **In ps output:** Running bash script

---

### Kill the Service

**Command:**
```bash
kill 1864
```

**Output:**
```
[1]+  Terminated                 ~/my-app/app.sh
```

The service stopped.

---

## Part 8: Capstone Challenge

### Challenge: Make Your App a Real systemd Service

In Day 3, you'll create a `.service` file so systemd manages your app automatically.

This means:
- Starts at boot
- Restarts if it crashes
- Logs are in `journalctl`
- Control with `systemctl start/stop/restart`

---

## Key Takeaways

✅ **systemd manages services** — start/stop/restart/enable/disable
✅ **journalctl shows logs** — debugging tool for service problems
✅ **OSI model (7 layers)** — understand where problems live
✅ **TCP = reliable, UDP = fast** — different tools for different jobs
✅ **DNS resolves names to IPs** — TTL controls caching
✅ **TLS encrypts + proves identity** — but certificates expire!
✅ **dig, ss, curl** — your debugging toolkit
✅ **You can create services** — background apps you can control

---

## What's Next: Week 1, Day 3

Tomorrow we'll do the **shell navigation & file management** we skipped, then jump into:
- Creating systemd `.service` files
- Making your app auto-restart
- Docker intro (containerization)

---

## Practice Commands Reference

```bash
# Service management
systemctl list-units --type=service --state=running
systemctl status cron.service
systemctl start cron.service
systemctl stop cron.service
systemctl restart cron.service
systemctl enable cron.service

# View logs
journalctl -u cron.service
journalctl -u cron.service -f    # follow (like tail -f)

# DNS queries
dig google.com
dig google.com MX
dig -x 142.251.36.206

# See what's listening
ss -tuln
ss -tuln | grep :3000    # find port 3000

# Make HTTP requests
curl https://google.com
curl -i https://google.com
curl -v https://google.com
curl -s -o /dev/null -w "%{http_code}\n" https://google.com

# Create and run a service
mkdir -p ~/my-app
cat > ~/my-app/app.sh << 'EOF'
#!/bin/bash
echo "App started at $(date)"
while true; do
  echo "$(date): App running..."
  sleep 5
done
EOF

chmod +x ~/my-app/app.sh
~/my-app/app.sh &
ps aux | grep app.sh
kill <PID>
```

---

## Reflection Questions

Before Day 3, ask yourself:

1. **What's the difference between SIGTERM and SIGKILL?** (Nice vs force)
2. **If a service keeps crashing, where do you check?** (journalctl logs)
3. **Why does DNS have a TTL?** (Cache control, speed vs updates)
4. **If you can't reach your server, which tool do you use?** (dig for DNS, ss for listening ports, curl for HTTP)
5. **What layer is HTTP?** (Layer 7, Application)
6. **What layer is TCP?** (Layer 4, Transport)

If you can answer these, **you're ready for Day 3.** 🚀

---

**You've completed Week 1, Day 2 like a pro.** Tomorrow: File management, then systemd service files. See you then! 💪
