
---

# 🔧 1. Website / App Is Down (Unreachable or “Connection Timed Out”)

### 🧠 The Logic

When a site doesn’t load, the issue can exist in **four main layers**:

1. **DNS** — domain doesn’t point to the right IP
2. **Network** — the server can’t be reached from the internet
3. **Service** — the server is up, but web service (nginx, node, etc.) is down
4. **Application** — backend is up, but app is erroring (e.g. database issue)

Understanding which layer is failing lets you **narrow scope fast**.

---

### 🔍 Step-by-Step

#### Step 1: DNS Resolution

Run:

```bash
dig +short example.com
```

If no result → DNS issue.
If IP appears → DNS is fine.

> Why: DNS translates a domain to an IP. If this fails, nothing else matters.

Then check propagation:

```bash
dig example.com @8.8.8.8
dig example.com @1.1.1.1
```

If different IPs → propagation delay or stale record.

---

#### Step 2: Network Reachability

```bash
ping -c 4 <ip>
traceroute <ip>
```

If no response → network or firewall issue.

> Why: If packets can’t reach the server, it’s likely blocked upstream (firewall, routing, cloud security group).

If the server is reachable, move to layer 3.

---

#### Step 3: Web Server Layer

SSH into the host and check:

```bash
sudo systemctl status nginx
sudo systemctl status apache2
```

If “inactive” or “failed” → restart:

```bash
sudo systemctl restart nginx
```

> Why: The web service might have crashed or failed to bind to port 80/443.

Check open ports:

```bash
sudo ss -tulpn | grep ':80\|:443'
```

If nothing is listening → service issue.

---

#### Step 4: Application Health

Try internal request:

```bash
curl -I http://localhost
```

If it works locally but fails remotely → firewall or proxy problem.
If it fails locally too → app or code issue.

---

#### 🧾 Collect for DevOps/NOC

* `dig` results
* `ping` / `traceroute` output
* `systemctl` status
* `curl -I` output
* Server name and time

#### 🚨 Escalation

* DNS failure → DNS / NOC team
* Network failure → NOC
* Service failure → DevOps
* App-level failure (500, timeout, DB error) → DevOps or developers

---

# 🔒 2. SSL / HTTPS Certificate Errors

### 🧠 The Logic

SSL ensures secure communication. Errors typically mean:

* The certificate is **expired**
* The **domain name doesn’t match** the certificate (mismatch)
* The certificate **chain** is broken or incomplete

---

### 🔍 Step-by-Step

#### Step 1: Browser Check

Open the site → click the 🔒 icon → “View certificate” → check expiry and common name (CN).

#### Step 2: Terminal Check

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

Shows:

```
notBefore=2024-01-01
notAfter=2025-01-01
subject=CN=example.com
```

> If expired or CN ≠ domain → that’s the issue.

#### Step 3: Full Certificate Chain

```bash
openssl s_client -showcerts -connect example.com:443
```

If intermediate certs missing → browsers may reject connection.

#### Step 4: Web Server Config

Check nginx/Apache SSL block:

```bash
sudo grep ssl_certificate /etc/nginx/sites-enabled/* 
```

> Sometimes points to the wrong `.crt` file.

---

#### 🧾 Collect for DevOps/NOC

* Browser screenshot
* Output of `openssl` commands
* Web server config path

#### 🚨 Escalation

* Expired → Renew via Let’s Encrypt / CA (NOC or DevOps)
* Mismatch → DevOps reconfigures cert paths
* Missing intermediate → upload full chain cert

---

# 🌐 3. DNS Issues (Domain Not Resolving or Old IP)

### 🧠 The Logic

DNS is like a phonebook: your browser asks, “What IP is google.com?”
If the record hasn’t updated, you’re calling the wrong number.

Common causes:

* Record not published on authoritative servers
* TTL delay
* DNS cache not cleared

---

### 🔍 Step-by-Step

#### Step 1: Check Current Record

```bash
dig example.com +noall +answer
```

Shows TTL and IP.

#### Step 2: Check Authoritative Servers

```bash
dig example.com +trace
```

Walks through the full resolution path → reveals which nameserver has the old value.

#### Step 3: Global Check

Use [DNSChecker.org](https://dnschecker.org)
If some regions show old IP → propagation delay.

---

#### 🧾 Collect for DevOps/NOC

* `dig +trace` output
* DNS provider screenshot (record + TTL)
* Propagation tool screenshot

#### 🚨 Escalation

* Wrong IP at DNS provider → NOC
* Propagation delay → monitor, inform client
* TTL too high → suggest lowering for future changes

---

# 🐳 4. Docker Container Won’t Start / Keeps Restarting

### 🧠 The Logic

Docker containers isolate applications.
If one won’t start, it’s usually because:

* The **app crashed** on boot
* **Missing environment variable / dependency**
* **Port conflict**
* **Wrong CMD or entrypoint**

---

### 🔍 Step-by-Step

#### Step 1: List Containers

```bash
docker ps -a
```

If “Exited (1)” or “Exited (137)” → crashed.

#### Step 2: Logs

```bash
docker logs <container>
```

Look for Python/Node errors or config file missing.

#### Step 3: Inspect

```bash
docker inspect <container> --format '{{.State.ExitCode}}'
```

Exit code hints at problem:

* 0 = clean exit
* 1 = app crash
* 137 = killed (OOM)
* 126/127 = permission or command missing

#### Step 4: Manual Run

```bash
docker run -it --rm <image> /bin/bash
```

Then manually run the app’s start command.

> This helps isolate if the entrypoint or environment is wrong.

---

#### 🧾 Collect for DevOps

* Container name, image tag
* `docker ps -a` output
* Last 100 lines of logs
* Exit code

#### 🚨 Escalation

* Crash due to code → DevOps or developer
* Missing env/config → DevOps updates secrets/config
* Port in use → NOC checks host services

---

# ☸️ 5. Kubernetes Pod in CrashLoopBackOff

### 🧠 The Logic

In Kubernetes, each pod is like a mini container environment.
CrashLoopBackOff means: “The pod started, crashed, restarted… repeatedly.”

Common causes:

* Application error on startup
* Bad config / missing environment variables
* Health probe failures
* Resource limits too low (OOMKilled)

---

### 🔍 Step-by-Step

#### Step 1: Check Pod Status

```bash
kubectl get pods -n <namespace>
```

If STATUS = `CrashLoopBackOff`, continue.

#### Step 2: Describe the Pod

```bash
kubectl describe pod <pod> -n <namespace>
```

Check “Events” section:

* `OOMKilled` → resource limit issue
* `ImagePullBackOff` → registry / credentials
* `Back-off restarting failed container` → app crash

#### Step 3: Check Logs

```bash
kubectl logs <pod> -n <namespace>
kubectl logs -p <pod> -n <namespace>    # previous attempt
```

If you see app-level error (e.g., “missing ENV variable”), that’s the root cause.

#### Step 4: Check Config & Resources

```bash
kubectl get deployment <name> -n <namespace> -o yaml | grep -A10 "resources:"
```

> Resource limits too small → OOMKilled
> Wrong image → pull failure

---

#### 🧾 Collect for DevOps

* `kubectl describe` and `kubectl logs` output
* Deployment version (image tag)
* Timestamps and recent rollouts

#### 🚨 Escalation

* CrashLoopBackOff with app logs → DevOps
* OOMKilled → DevOps or infra to adjust limits
* ImagePullBackOff → check registry credentials

---

# 💽 6. Server Performance Degradation (High CPU / Memory)

### 🧠 The Logic

High usage usually means:

* Application is stuck in a loop (CPU)
* Memory leak (container or process)
* Disk full (logs not rotating)
* Too many pods/containers on one node

---

### 🔍 Step-by-Step

#### Step 1: Check Top Processes

```bash
top -o %CPU
```

> Find which process is consuming resources.

#### Step 2: Disk Space

```bash
df -h
du -sh /var/log/*
```

> Full disk causes many services to crash (no write space).

#### Step 3: System Logs

```bash
journalctl -xe --since "10 min ago"
```

> Look for kernel OOM messages or service restarts.

#### Step 4: Docker Stats (if containerized)

```bash
docker stats
```

#### Step 5: Kubernetes Nodes

```bash
kubectl top node
kubectl top pod -A
```

> Identify which pod is using excess resources.

---

#### 🧾 Collect for DevOps/NOC

* Screenshots of top/df
* Log snippets
* `kubectl top` output
* Time of spike

#### 🚨 Escalation

* App-level leak → DevOps
* Disk issue → NOC to clean/rotate logs
* Node resource exhaustion → NOC / infra scaling

---

# ⚡ BONUS: Incident Response Flow (How to Think During Any Problem)

1. **Identify scope:** one user? one region? whole system?
2. **Reproduce:** confirm the issue yourself (never rely only on report).
3. **Check basics:** DNS → network → service → app → database.
4. **Collect facts:** logs, commands, timestamps.
5. **Communicate clearly:**

   * “Impact: X users affected.”
   * “Cause (tentative): DNS not propagating.”
   * “Next step: DevOps renewing SSL cert.”
6. **Update clients with facts only**, no guesses.
7. **Document RCA (root cause analysis)** after resolution.

---

Would you like me to continue this and create:

* A **printable “Incident Playbook” PDF** (with commands + logic summary), or
* An **interactive “lab-style” tutorial** where you can actually simulate and fix each problem (DNS fail, SSL expire, pod crash, etc.) using sample commands?

That way you can practice and be 100% confident in live situations.
