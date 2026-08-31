Perfect! Let me break down the **Week 1 Mastery Checklist** so it sticks:

---

## **Week 1 Mastery Checklist (0/8)**

### **1. ✅ Navigate, search, and edit files entirely from the shell**

**What it means:**
- No GUI file manager
- Use `ls`, `find`, `grep`, `cat`, `nano`/`vim`
- Move around with `cd`, see where you are with `pwd`

**You did this:**
```bash
find ~ -name "access.log"          # Found your log file
grep "500" access.log              # Searched for errors
cat Dockerfile                     # Viewed file contents
```

**Why it matters:** Backend work happens in terminals. GUI is slow. You need speed.

---

### **2. ✅ Understand Linux permissions and the difference between a file's owner and a process's user**

**What it means:**
- Know `rwxr-xr-x` (owner/group/others, read/write/execute)
- Understand that a **file owner** (user) is different from a **process owner**
- File might be owned by `root`, but `apache` process runs it

**You learned:**
```
-rw-r--r-- 1 umer_muhammed umer_muhammed  807 Aug 22 13:36 .profile
 │  │  │                    │                │
 │  │  │                    │                └─ File contents
 │  │  └─ Others: read only (r--)
 │  └───── Group: read only (r--)
 └──────── Owner: read & write (rw-)
```

**Why it matters:** 90% of "permission denied" errors come from not understanding this. Knowing it saves hours of debugging.

---

### **3. ✅ Find and kill a runaway process and read a service's logs with journalctl**

**What it means:**
- Use `ps aux` to find processes
- Use `kill` (SIGTERM) or `kill -9` (SIGKILL) to stop them
- Use `journalctl` to see what went wrong

**You did this:**
```bash
ps aux | grep app.sh               # Found your app (PID 1864)
kill 1864                          # Killed it gracefully
journalctl -u myapp.service -f     # Followed live logs
```

**Why it matters:** Production apps crash. You need to kill them and read logs to fix it.

---

### **4. ✅ Chain commands with pipes to slice logs and API output in one line**

**What it means:**
- Use `|` to connect tools
- One line solves problems that would take scripts in other languages
- Mastery: `cat log.txt | grep "500" | wc -l` answers "how many 500 errors?"

**You did this:**
```bash
cat access.log | grep "GET" | grep "200" | wc -l    # Count GET 200s
ps aux | grep python | wc -l                        # Count Python processes
cat status.log | cut -d' ' -f5 | sort | uniq -c     # Count status codes
```

**Why it matters:** This is the Unix philosophy. Small tools, powerful combined. You just solved production problems in one line.

---

### **5. ✅ Explain the OSI/TCP-IP layers and when to use TCP versus UDP**

**What it means:**
Know the 7 layers:
```
Layer 7: HTTP, HTTPS, DNS (Application)
Layer 6: Encryption formatting
Layer 5: Session management
Layer 4: TCP (reliable, ordered) vs UDP (fast, lossy)
Layer 3: IP addresses
Layer 2: MAC addresses
Layer 1: Physical cables
```

**TCP vs UDP:**
- **TCP** = ordered, reliable (email, databases, HTTP) — slower but guaranteed
- **UDP** = fast, lossy (DNS, video, gaming) — faster but can lose packets

**Why it matters:** When debugging "server is slow," you need to know which layer is the problem. TCP adds latency but guarantees delivery. DNS uses UDP because one lost packet doesn't matter.

---

### **6. ✅ Trace a request through DNS, TCP, TLS, and HTTP and name the tool to inspect each**

**What it means:**
When someone says "my app is down," you can explain:

**Layer 3 (DNS):** `dig google.com` → Is DNS working?
**Layer 4 (TCP):** `nc -zv google.com 443` → Can I connect to port 443?
**Layer 7 (TLS):** `openssl s_client -connect google.com:443` → Is certificate valid?
**Layer 7 (HTTP):** `curl https://google.com` → What status code?

**You did this:**
```bash
~/trace.sh google.com
# [1] DNS ✓
# [2] TCP ✓
# [3] TLS ✓
# [4] HTTP ✓
```

**Why it matters:** Each layer can fail independently. You isolate which one broke instead of guessing.

---

### **7. ✅ Use dig, ss, curl, and tcpdump to diagnose a networking problem**

**What it means:**
Know your debugging toolkit:

| Tool | Does | Example |
|------|------|---------|
| `dig` | DNS lookup | `dig google.com` → 142.251.38.14 |
| `ss` | See listening ports | `ss -tlnp` → Port 3000 listening |
| `curl` | HTTP request | `curl -v https://google.com` → See TLS + HTTP |
| `tcpdump` | Watch packets | `sudo tcpdump port 443` → See TCP + TLS |

**You did this:**
```bash
dig google.com                     # DNS works
ss -tlnp                           # Check what's listening
curl -v https://google.com         # See full handshake
sudo tcpdump -n port 443           # Watch packets
```

**Why it matters:** When something breaks, you have tools to inspect each layer. No guessing.

---

### **8. ✅ Write a script that resolves DNS, tests a connection, and checks a cert's expiry**

**What it means:**
You built `trace.sh` that:
1. Resolves DNS with `dig`
2. Tests TCP with `nc`
3. Checks certificate with `openssl`
4. Checks HTTP status with `curl`
5. Reports pass/fail clearly

**You did this:**
```bash
~/trace.sh google.com
~/trace.sh example.com
~/trace.sh doesnotexist.com  # Fails at DNS
```

**Why it matters:** This is a real production tool. You've automated layer-by-layer debugging. This script could catch real outages.

---

## **Visual Summary**

```
┌─────────────────────────────────────────────┐
│ Week 1: From Zero to Production-Ready       │
├─────────────────────────────────────────────┤
│ Day 1: Linux fundamentals & pipes           │
│ ✓ Navigate files (shell mastery)            │
│ ✓ Understand permissions (owner/group)      │
│ ✓ Chain commands with pipes                 │
│                                              │
│ Day 2: Services & networking                │
│ ✓ Manage processes (kill, find)             │
│ ✓ Read logs with journalctl                 │
│ ✓ Know OSI layers & TCP vs UDP              │
│                                              │
│ Day 3: Debugging & capstone                 │
│ ✓ Trace DNS → TCP → TLS → HTTP              │
│ ✓ Use dig, ss, curl, tcpdump                │
│ ✓ Built trace.sh script                     │
│                                              │
│ RESULT: You can debug like a senior         │
│ engineer — isolate failures at each layer   │
└─────────────────────────────────────────────┘
```

---

## **The Real Achievement**

Before Week 1, if someone said "my website is down," you'd panic and restart everything.

**After Week 1:**
```
1. Is DNS working? → dig
2. Is TCP connecting? → nc
3. Is cert valid? → openssl
4. Is HTTP responding? → curl
5. Which layer failed? → Fix that layer specifically

Result: You just diagnosed a production outage in 5 minutes.
```

---

## **Checklist Tracker**

Print this and check off as you verify:

```
Week 1 Mastery (check yourself)

□ Can I navigate and search files from shell only?
  Test: find ~ -name "*.sh" && grep something file && cat file

□ Do I understand rwxr-xr-x and owner vs user?
  Test: ls -la | explain what each part means

□ Can I find, kill, and read logs?
  Test: ps aux, kill, journalctl -u service

□ Can I pipe 3+ tools into one line?
  Test: ps aux | grep python | wc -l

□ Can I explain OSI layers and TCP vs UDP?
  Test: Explain out loud (TCP ordered/reliable, UDP fast/lossy)

□ Can I trace DNS → TCP → TLS → HTTP?
  Test: Run ~/trace.sh google.com, explain each step

□ Do I know dig, ss, curl, tcpdump?
  Test: Use each one on a real host

□ Did I write and test trace.sh?
  Test: Run it on working, broken, and nonexistent hosts
```

---

**You're now at 8/8 on this checklist!** 🎉

**Next: Either Docker today, or Week 2 databases tomorrow. Your call!** 🚀
