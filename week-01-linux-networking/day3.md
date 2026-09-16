# Week 1, Day 3: Shell Mastery & Networking Tools (Capstone)

## Overview
**Goal:** Master the shell + networking tools to isolate failures at each layer (DNS → TCP → TLS → HTTP).

**What you'll build:** A `trace.sh` script that diagnoses network issues layer-by-layer, exactly like a senior engineer would.

---

## Part 1: Shell Mastery Verification

You should be comfortable with these commands by now:

### Finding Files
```bash
find ~ -name "*.sh" -type f
```
**Output:** Lists all bash scripts in your home directory
```
/home/umer_muhammed/my-app/app.sh
/home/umer_muhammed/trace.sh
```

### Grepping Logs
```bash
grep '500' access.log | head -5
```
**Output:** First 5 lines containing "500"
```
2026-8-30 10:16:36 GET /api/users 500
```

### Piping Multiple Tools
```bash
ps aux | grep python | wc -l
```
**Output:** Count of Python processes
```
3
```

**These are the fundamentals.** You're now comfortable in the terminal — this is the gateway to backend mastery.

---

## Part 2: Networking Tools Deep Dive

### DNS Resolution: `dig`

**What it does:** Resolves domain names to IP addresses (Layer 3 - Network)

```bash
dig google.com
```

**Output:**
```
; <<>> DiG 9.20.18-1ubuntu2-Ubuntu <<>> google.com
;; ANSWER SECTION:
google.com.             46      IN      A       142.251.38.14
;; Query time: 156 msec
;; SERVER: 10.255.255.254#53 (UDP)
```

**Breaking it down:**
- `google.com` → `142.251.38.14` (the IP address)
- TTL = 46 seconds (cache this for 46 seconds)
- Query took 156 milliseconds
- Used UDP port 53 (standard DNS)

**Short form (just the IP):**
```bash
dig +short google.com
```
**Output:** `142.251.38.14`

---

### TCP Connection Test: `nc` (netcat)

**What it does:** Test if a TCP port is open (Layer 4 - Transport)

```bash
nc -zv google.com 443
```

**Output (success):**
```
Connection to google.com 443 port [tcp/https] succeeded!
```

**Output (failure):**
```
nc: connect to google.com port 443 (tcp) failed: Connection refused
```

**Flags:**
- `-z` = just check, don't send data
- `-v` = verbose (show what's happening)

---

### TLS Certificate Inspection: `openssl`

**What it does:** Check SSL/TLS certificate details (Layer 7 - Application/Security)

**Check certificate expiry:**
```bash
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -enddate
```

**Output:**
```
notAfter=Nov  2 08:37:34 2026 GMT
```

**Meaning:** Certificate expires November 2, 2026. Still valid.

**Breaking it down:**
- `openssl s_client -connect google.com:443` = Connect to server and get certificate
- `openssl x509 -noout -enddate` = Extract just the expiry date
- `2>/dev/null` = Hide errors

---

### HTTP Status Code: `curl`

**What it does:** Make HTTP requests and inspect responses (Layer 7 - Application)

**Get just the status code:**
```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://google.com
```

**Output:**
```
301
```

**Meaning:** HTTP 301 Moved Permanently (redirect to www.google.com)

**Verbose mode (see full handshake):**
```bash
curl -v https://google.com
```

**Output shows:**
```
* Host google.com:443 was resolved.
* IPv4: 142.251.38.14
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS change cipher, Change cipher spec (1):
< HTTP/1.1 301 Moved Permanently
< location: https://www.google.com/
```

This shows the **entire TLS handshake and HTTP exchange**.

---

## Part 3: Build trace.sh — The Capstone

### The Script
```bash
cat > ~/trace.sh << 'EOF'
#!/bin/bash

# Trace a host through all network layers
# Usage: ./trace.sh google.com

HOST=$1

if [ -z "$HOST" ]; then
  echo "Usage: $0 <host>"
  exit 1
fi

echo "=== Tracing $HOST ==="
echo

# Step 1: DNS Resolution (Layer 3)
echo "[1] DNS Resolution"
IP=$(dig +short "$HOST" A | head -1)
if [ -z "$IP" ]; then
  echo "❌ FAIL: DNS did not resolve"
  exit 1
else
  echo "✓ PASS: $HOST -> $IP"
fi
echo

# Step 2: TCP Connection (Layer 4)
echo "[2] TCP Connection to port 443"
if nc -zv "$HOST" 443 2>&1 | grep -q "succeeded"; then
  echo "✓ PASS: TCP connection successful"
else
  echo "❌ FAIL: TCP connection failed"
  exit 1
fi
echo

# Step 3: TLS Certificate (Layer 7 - Security)
echo "[3] TLS Certificate Expiry"
EXPIRY=$(echo | openssl s_client -connect "$HOST:443" 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
if [ -z "$EXPIRY" ]; then
  echo "❌ FAIL: Could not read certificate"
  exit 1
else
  echo "✓ PASS: Certificate expires: $EXPIRY"
fi
echo

# Step 4: HTTP Status (Layer 7 - Application)
echo "[4] HTTP Status Code"
STATUS=$(curl -sS -o /dev/null -w "%{http_code}" "https://$HOST")
if [ "$STATUS" = "200" ] || [ "$STATUS" = "301" ] || [ "$STATUS" = "302" ] || [ "$STATUS" = "307" ]; then
  echo "✓ PASS: HTTP $STATUS"
else
  echo "❌ FAIL: HTTP $STATUS (unexpected)"
  exit 1
fi
echo

echo "=== All checks passed for $HOST ==="
EOF
```

Make it executable:
```bash
chmod +x ~/trace.sh
```

---

### Testing trace.sh

**Test 1: Working host**
```bash
~/trace.sh google.com
```

**Output:**
```
=== Tracing google.com ===
[1] DNS Resolution
✓ PASS: google.com -> 142.251.38.14
[2] TCP Connection to port 443
✓ PASS: TCP connection successful
[3] TLS Certificate Expiry
✓ PASS: Certificate expires: Nov  2 08:37:34 2026 GMT
[4] HTTP Status Code
✓ PASS: HTTP 301
=== All checks passed for google.com ===
```

**What this proves:**
- DNS works ✓
- TCP port 443 is open ✓
- Certificate is valid (expires Nov 2) ✓
- Server responds with HTTP 301 ✓

---

**Test 2: Another working host**
```bash
~/trace.sh example.com
```

**Output:**
```
=== Tracing example.com ===
[1] DNS Resolution
✓ PASS: example.com -> 172.66.147.243
[2] TCP Connection to port 443
✓ PASS: TCP connection successful
[3] TLS Certificate Expiry
✓ PASS: Certificate expires: Oct 27 22:17:21 2026 GMT
[4] HTTP Status Code
✓ PASS: HTTP 200
=== All checks passed for example.com ===
```

---

### Test 3: Breaking It — DNS Failure
```bash
~/trace.sh doesnotexist12345.com
```

**Output:**
```
=== Tracing doesnotexist12345.com ===
[1] DNS Resolution
❌ FAIL: DNS did not resolve
```

**Insight:** Script stops at Layer 3. DNS can't resolve. No point checking TCP/TLS/HTTP.

**This is professional debugging** — isolate the layer that failed.

---

## Part 4: Watching Packets with tcpdump

### Install tcpdump
```bash
sudo apt install tcpdump -y
```

### See TLS Handshake on the Wire
```bash
sudo tcpdump -n port 443
```

In another terminal:
```bash
curl -v https://google.com
```

**You'll see packets:**
```
IP <your-ip> > 142.251.38.14.443: Flags [S], seq 0       # TCP SYN
IP 142.251.38.14.443 > <your-ip>: Flags [S.], seq ...    # TCP SYN-ACK
IP <your-ip> > 142.251.38.14.443: Flags [.], seq ...     # TCP ACK
(TLS handshake data)
(HTTP request/response data)
```

**This is the TCP 3-way handshake and TLS encryption happening in real-time.**

---

## Part 5: Interview Mastery — The Complete Picture

### The Question
> "What happens when you type `curl https://google.com` and hit enter?"

### Your Answer (Layer by Layer)

**Layer 3 — Network (DNS):**
- Query: `dig google.com`
- Result: `google.com → 142.251.38.14`
- If fails: "DNS didn't resolve. Check `dig`."

**Layer 4 — Transport (TCP):**
- Query: `nc -zv google.com 443`
- Result: "Connection succeeded" to port 443
- If fails: "TCP port 443 is closed. Server is down or blocking HTTPS."

**Layer 7 — TLS (Encryption + Identity):**
- Query: `openssl s_client -connect google.com:443`
- Result: Certificate is valid until Nov 2, 2026
- If fails: "Certificate expired. Web server needs new cert. THIS BREAKS THE ENTIRE SITE."

**Layer 7 — HTTP (Application):**
- Query: `curl https://google.com`
- Result: HTTP 301 (redirect to www.google.com)
- If fails: "Server not responding. Check application logs with `systemctl status` or `journalctl -u myapp`."

---

### Common Pitfalls (Don't Fall Into These)

❌ **Pitfall 1:** Assume "server is down" when DNS didn't resolve
- **Fix:** Always test each layer independently. Run `dig` first.

❌ **Pitfall 2:** Run everything as root with `chmod 777`
- **Why:** Hides real permission bugs. Security hole. Never do this.
- **Fix:** Understand file permissions. Use `chmod 755` for executables, `644` for files.

❌ **Pitfall 3:** Ignore stderr. "Script produces no output"
- **Why:** You're probably only seeing stdout. Errors go to stderr.
- **Fix:** Redirect with `2>&1`: `command 2>&1 | grep error`

❌ **Pitfall 4:** Forget TCP vs UDP matter
- **Why:** Using TCP where UDP fits wastes latency. UDP is fast but loses packets.
- **Fix:** Know when to use each. DNS = UDP. SSH = TCP.

---

## Key Takeaways

✅ **Shell mastery** — find, grep, pipes, three tools in one line
✅ **Networking tools** — dig (DNS), nc (TCP), openssl (TLS), curl (HTTP)
✅ **Layer isolation** — Debug each layer independently
✅ **trace.sh script** — Automate the debugging process
✅ **Packet inspection** — See TLS handshake on the wire with tcpdump
✅ **Interview ready** — Explain what happens at each layer when a request fails

---

## Commands Reference

```bash
# DNS
dig google.com
dig +short google.com
dig google.com MX

# TCP
nc -zv google.com 443
ss -tlnp

# TLS
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -enddate

# HTTP
curl -sS -o /dev/null -w "%{http_code}\n" https://google.com
curl -v https://google.com

# Packets
sudo tcpdump -n port 443

# Your script
~/trace.sh google.com
~/trace.sh example.com
~/trace.sh doesnotexist.com
```

---

## What's Next: Week 2

Now that you understand:
- ✅ Linux fundamentals
- ✅ Processes & signals
- ✅ Services with systemd
- ✅ Networking layers & tools

**Week 2 will cover:**
- Databases (PostgreSQL, SQL basics)
- REST APIs (building your own)
- Authentication (JWT, sessions)

But first, we'll **finish Docker** (the capstone from Day 3) so you can package everything.

---

## Practice Exercises

1. **Debug a failing service:**
   - Start your `myapp.service` (from Day 2)
   - Run `~/trace.sh localhost` (won't work, but try)
   - Use the tools to diagnose why

2. **Inspect a site:**
   ```bash
   ~/trace.sh github.com
   ~/trace.sh stackoverflow.com
   ~/trace.sh your-own-domain.com
   ```

3. **Certificate expiry alert:**
   - Write a script that checks certificate expiry
   - Use `openssl` to extract the date
   - Alert if expiring within 30 days

4. **Tcpdump analysis:**
   - Run `sudo tcpdump -n port 443` while making requests
   - Identify the TCP SYN, SYN-ACK, ACK
   - Observe TLS encrypted data flowing

---

**You've completed Week 1.** You now understand Linux, services, networking, and how to debug like a senior engineer. 🚀
