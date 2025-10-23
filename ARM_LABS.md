
---

# 🧪 ARM Technical Troubleshooting Labs

**Focus:** Ubuntu + Docker + Kubernetes + DNS + SSL
**Goal:** Practice real investigation logic, not just memorize commands.

---

## 🧩 Lab 1 – DNS Resolution Failure

**Scenario:**
A client reports: “Our sportsbook site (`betzone365.com`) isn’t loading.”
You ping it — no response. Let’s troubleshoot.

### 🔧 Objective:

Identify whether the issue is with DNS, networking, or the app.

---

### 🧠 Step-by-step reasoning and practice:

#### 1️⃣ Check DNS resolution:

```bash
dig +short betzone365.com
```

**Possible results:**

* **No output:** Domain not resolving → DNS issue.
* **Old IP address:** DNS change not propagated.
* **Correct IP:** DNS okay → check connectivity next.

#### 2️⃣ Check which nameserver is giving the wrong IP:

```bash
dig betzone365.com +trace
```

**What you learn:**
This shows the path from root → TLD → authoritative server.
If an old IP appears at the last step → the zone file is outdated.

#### 3️⃣ Confirm propagation globally:

Use [https://dnschecker.org](https://dnschecker.org)

#### 🧾 Collect:

* `dig +trace` output
* IP comparison (old vs new)
* DNS provider screenshot

#### ✅ Resolution logic:

If DNS is outdated, escalate to NOC to refresh zone or reduce TTL.
Communicate to client:

> “The domain is still propagating globally. It can take up to the TTL period (e.g., 2 hours).”

---

## 🔒 Lab 2 – SSL Certificate Expired

**Scenario:**
Browser shows:

> “Your connection is not private. NET::ERR_CERT_DATE_INVALID.”

You must confirm expiry and who renews it.

---

### 🔧 Objective:

Verify SSL expiry and diagnose cause.

#### 1️⃣ Check expiry:

```bash
openssl s_client -connect betzone365.com:443 -servername betzone365.com </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
```

You’ll see:

```
notBefore=Mar  1 00:00:00 2024 GMT
notAfter=Oct 21 23:59:59 2025 GMT
subject=CN=betzone365.com
```

If `notAfter` < today → certificate expired.

#### 2️⃣ Check cert chain:

```bash
openssl s_client -showcerts -connect betzone365.com:443
```

Missing intermediates? Some browsers reject incomplete chains.

#### 3️⃣ Inspect server config:

```bash
sudo grep ssl_certificate /etc/nginx/sites-enabled/*
```

#### 🧾 Collect:

* Certificate subject + expiry
* Screenshot of browser error
* `openssl` output

#### ✅ Resolution logic:

If expired, Let’s Encrypt auto-renew may have failed — NOC or DevOps renews.
If mismatch → wrong cert deployed to the domain.
Inform client:

> “The SSL certificate expired; renewal is in progress. It will update automatically within 10–15 minutes.”

---

## 🐧 Lab 3 – Ubuntu Web Server Down

**Scenario:**
NOC alert: “Site health check failing on port 80.”
You need to see if the service or system is the problem.

---

### 🔧 Objective:

Determine whether the web service crashed or the system is overloaded.

#### 1️⃣ Check if nginx is running:

```bash
sudo systemctl status nginx
```

If you see:

```
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
   Active: failed (Result: exit-code)
```

→ The service crashed.

#### 2️⃣ Restart service:

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

#### 3️⃣ Check logs:

```bash
journalctl -u nginx -n 50
```

Look for errors like “bind() to 0.0.0.0:80 failed” → port in use.

#### 4️⃣ Identify what’s using port 80:

```bash
sudo ss -tulpn | grep :80
```

#### 🧾 Collect:

* `systemctl` output
* Log snippet
* `ss` output

#### ✅ Resolution logic:

* Port conflict → stop old service
* Config error → fix syntax & restart
* High load → escalate to DevOps for scaling

---

## 🐳 Lab 4 – Docker Container Crashes Immediately

**Scenario:**
You deploy a new container, but it keeps exiting with status `(1)`.

---

### 🔧 Objective:

Find out **why** it’s failing and what logs show.

#### 1️⃣ List containers:

```bash
docker ps -a
```

Output:

```
CONTAINER ID   IMAGE             STATUS
a1b2c3d4e5     sportsbook:1.2    Exited (1) 5s ago
```

#### 2️⃣ Check logs:

```bash
docker logs a1b2c3d4e5
```

If you see:

```
Error: Missing ENV variable DATABASE_URL
```

→ Misconfigured environment.

#### 3️⃣ Check container details:

```bash
docker inspect a1b2c3d4e5 --format '{{.State.ExitCode}}'
```

#### 4️⃣ Run container manually:

```bash
docker run -it --rm sportsbook:1.2 /bin/bash
```

Try to start app manually:

```bash
python app.py
```

If it fails, you can see the missing dependencies or files.

#### 🧾 Collect:

* Logs
* Exit code
* Env vars in run command

#### ✅ Resolution logic:

Missing configs → fix `.env` or Kubernetes ConfigMap.
App crash → forward logs to DevOps.
Port in use → check host or compose setup.

---

## ☸️ Lab 5 – Kubernetes Pod CrashLoopBackOff

**Scenario:**
After a deployment, a pod keeps restarting.

---

### 🔧 Objective:

Identify what’s causing repeated crashes.

#### 1️⃣ List pods:

```bash
kubectl get pods -n sportsbook
```

Output:

```
NAME                       READY   STATUS             RESTARTS   AGE
api-56789d4f7f-m8sqr       0/1     CrashLoopBackOff   5          3m
```

#### 2️⃣ Describe the pod:

```bash
kubectl describe pod api-56789d4f7f-m8sqr -n sportsbook
```

In **Events**, you might see:

```
Back-off restarting failed container
Last State: OOMKilled
```

#### 3️⃣ Get logs:

```bash
kubectl logs api-56789d4f7f-m8sqr -n sportsbook
kubectl logs -p api-56789d4f7f-m8sqr -n sportsbook
```

If you see:

```
Error: cannot connect to PostgreSQL
```

→ Dependency issue.

#### 4️⃣ Check resource limits:

```bash
kubectl get deployment api -n sportsbook -o yaml | grep -A10 resources:
```

#### 🧾 Collect:

* Describe + logs
* Deployment image tag
* Time of deployment

#### ✅ Resolution logic:

* `OOMKilled` → increase memory limit
* `ImagePullBackOff` → wrong registry or credentials
* App crash → misconfig or missing service (DB)

---

## 💽 Lab 6 – High CPU / Memory on Node

**Scenario:**
Grafana alert: “Node cpu > 90% for 10m”.

---

### 🔧 Objective:

Find which process/pod is the culprit.

#### 1️⃣ On Ubuntu host:

```bash
top -o %CPU
```

Find high usage process (e.g., `java` or `node`).

#### 2️⃣ Check Docker containers:

```bash
docker stats
```

High `%CPU` → note container ID.

#### 3️⃣ Check Kubernetes usage:

```bash
kubectl top node
kubectl top pod -A | sort -k3 -nr | head
```

See which pod is consuming most resources.

#### 4️⃣ Inspect that pod:

```bash
kubectl describe pod <pod> -n <ns>
```

#### 🧾 Collect:

* Top output
* `kubectl top pod`
* Logs of heavy pod

#### ✅ Resolution logic:

If one app/pod spikes → DevOps to profile and fix.
If node consistently full → scale up cluster or move workloads.

---

# 🧠 Practice Framework for Every Lab

Each lab trains the same **mental sequence**:

| Step                 | Question                    | Example Command                           |
| -------------------- | --------------------------- | ----------------------------------------- |
| 1️⃣ Observe          | What exactly is failing?    | “Site not loading”                        |
| 2️⃣ Reproduce        | Can I confirm it?           | `curl`, browser, ping                     |
| 3️⃣ Isolate Layer    | DNS? Network? Service? App? | `dig`, `ping`, `systemctl`                |
| 4️⃣ Collect Evidence | Facts only                  | logs, outputs, timestamps                 |
| 5️⃣ Communicate      | Clear escalation            | “Service down, nginx failed at 14:32 UTC” |

---

# ⚙️ Optional: Mini Project (for practice)

You can **build your own mini-lab** on a single VM or laptop using Docker and Minikube:

1. Install Docker & Minikube
2. Deploy a simple Nginx web app container
3. Manually:

   * Stop nginx to simulate downtime
   * Rename `.crt` file to simulate SSL failure
   * Delete DNS record (if using custom domain)
   * Create pod with invalid env var to simulate CrashLoopBackOff
4. Practice using the commands from each lab to diagnose

This simulation will make you **interview-proof** and **production-ready**.

---

Would you like me to generate the **lab environment setup guide** (how to install Docker, Minikube, and simulate each failure) — so you can *actually run* these scenarios locally or on a test server?
