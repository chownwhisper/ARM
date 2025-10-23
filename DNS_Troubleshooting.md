---

# 🌐 DNS TROUBLESHOOTING PLAYBOOK — ADVANCED VERSION

*(Step-by-step diagnostic path for NOC/ARM/DevOps)*

|  🧭 Step | 🎯 Purpose / Question               | 💻 Command                                                                                                                       | 🧾 Expected Output                          | 🔄 If Unexpected → Next Action               | 🧠 Why / Explanation                                                         |                                                       |
| :------: | :---------------------------------- | :------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------ | :------------------------------------------- | :--------------------------------------------------------------------------- | ----------------------------------------------------- |
|  **1️⃣** | Check if DNS resolves locally       | `dig example.com +short`                                                                                                         | ✅ `93.184.216.34`                           | ⚠️ No result → Step 2                        | Tests the resolver defined in `/etc/resolv.conf`. Verifies basic resolution. |                                                       |
|  **2️⃣** | Query Google Public DNS             | `dig example.com @8.8.8.8 +short`                                                                                                | ✅ IP address returned                       | ⚠️ No answer → Step 3                        | Confirms external DNS reachability. 8.8.8.8 is reliable reference.           |                                                       |
|  **3️⃣** | Cross-check Cloudflare DNS          | `dig example.com @1.1.1.1 +short`                                                                                                | ✅ Same IP as Step 2                         | ⚠️ Different IP → Step 4                     | Detects propagation delay (inconsistent DNS caches).                         |                                                       |
|  **4️⃣** | Confirm with multiple resolvers     | `for i in 8.8.8.8 1.1.1.1 9.9.9.9; do dig example.com @$i +short; done`                                                          | ✅ Same IP from all                          | ⚠️ Differences → Step 5                      | Verifies global resolver sync. Different results = propagation in progress.  |                                                       |
|  **5️⃣** | Identify authoritative name servers | `dig NS example.com +short`                                                                                                      | ✅ e.g. `ns1.digitalocean.com.`              | ❌ Empty → misconfigured domain               | Shows who owns the official DNS zone.                                        |                                                       |
|  **6️⃣** | Query authoritative server directly | `dig example.com @ns1.digitalocean.com +short`                                                                                   | ✅ Correct IP (final source of truth)        | ❌ No record → Step 7                         | Confirms if zone is correctly configured at the origin.                      |                                                       |
|  **7️⃣** | Inspect TTL (cache lifetime)        | `dig example.com +noall +answer`                                                                                                 | e.g. `example.com. 3600 IN A 93.184.216.34` | ⚠️ TTL too high (≥86400) → consider lowering | TTL tells how long old data persists in DNS caches.                          |                                                       |
|  **8️⃣** | Trace full DNS path                 | `dig +trace example.com`                                                                                                         | ✅ Ends with IP answer                       | ❌ Stops before final → Step 9                | Reveals delegation chain: Root → TLD → NS → Answer.                          |                                                       |
|  **9️⃣** | Verify delegation at registrar      | `whois example.com                                                                                                               | grep "Name Server"`                         | ✅ NS match Step 5                            | ❌ Different NS → update registrar                                            | Mismatch = broken DNS delegation, zone won’t resolve. |
|  **🔟**  | Clear local cache                   | **Linux:** `sudo systemd-resolve --flush-caches`<br>**Mac:** `sudo dscacheutil -flushcache`<br>**Windows:** `ipconfig /flushdns` | ✅ “Cache flushed”                           | ⚠️ Still cached → Step 11                    | Ensures your machine isn’t using stale DNS entries.                          |                                                       |
| **11️⃣** | Check DNS status code               | `dig example.com` → look for `status:` line                                                                                      | ✅ `NOERROR`                                 | ❌ `NXDOMAIN` → domain not registered         | `NXDOMAIN` = domain missing; `SERVFAIL` = upstream error.                    |                                                       |
| **12️⃣** | Inspect MX records (mail routing)   | `dig MX example.com +short`                                                                                                      | ✅ e.g. `10 mail.example.com.`               | ❌ None → misconfigured mail DNS              | Required for email delivery.                                                 |                                                       |
| **13️⃣** | Inspect CNAME alias                 | `dig CNAME www.example.com +short`                                                                                               | ✅ e.g. `example.com.`                       | ⚠️ None → check A record instead             | Verifies alias redirection (common for subdomains).                          |                                                       |
| **14️⃣** | Inspect TXT records (SPF/DKIM)      | `dig TXT example.com +short`                                                                                                     | ✅ `"v=spf1 include:_spf.google.com ~all"`   | ⚠️ Missing → incomplete domain setup         | Needed for email auth and domain verification.                               |                                                       |
| **15️⃣** | Check reverse DNS (PTR)             | `dig -x 93.184.216.34 +short`                                                                                                    | ✅ `example.com.`                            | ⚠️ None → missing PTR                        | Reverse lookup must match forward DNS for mail servers.                      |                                                       |
| **16️⃣** | Verify global propagation           | Check [https://dnschecker.org](https://dnschecker.org)                                                                           | ✅ Same IP worldwide                         | ⚠️ Mixed results → wait for TTL expiry       | Confirms global cache updates.                                               |                                                       |
| **17️⃣** | Confirm the IP serves traffic       | `curl -I http://93.184.216.34`                                                                                                   | ✅ `HTTP/1.1 200 OK`                         | ❌ Timeout → Step 18                          | Ensures the target service is actually running.                              |                                                       |
| **18️⃣** | Test connectivity                   | `ping example.com` → `traceroute example.com`                                                                                    | ✅ Normal path                               | ⚠️ Drop mid-route → network/firewall issue   | Confirms whether the host is reachable.                                      |                                                       |
| **19️⃣** | If DNS OK but site fails            | `nslookup example.com` → `curl -v example.com`                                                                                   | ✅ Resolves + connects                       | ⚠️ DNS OK but no response → web/app issue    | Confirms DNS is healthy, shifts focus to HTTP/SSL.                           |                                                       |
| **20️⃣** | Verify issue resolution             | Repeat `dig example.com @8.8.8.8`, `@1.1.1.1`                                                                                    | ✅ Identical IPs from all                    | ✅ Incident closed                            | Confirms DNS propagation is complete.                                        |                                                       |

---

## 🧩 Visual Logic Flow (Simplified)

```text
No IP? → dig @8.8.8.8
│
├─ No → dig @1.1.1.1 → compare → if different → check TTL
│
├─ Still fails → dig NS → query authoritative → whois check
│
├─ Works locally but not globally → TTL or propagation
│
└─ DNS OK but site down → curl / traceroute / app-level issue
```

---

## 🧠 Pro Tips for ARM/NOC Work

| Scenario                            | Quick Diagnostic                      | Likely Root Cause                   |
| ----------------------------------- | ------------------------------------- | ----------------------------------- |
| DNS resolves differently in regions | `dig @8.8.8.8` vs `@1.1.1.1`          | Propagation delay or stale caches   |
| `SERVFAIL` on all resolvers         | `dig +trace` stops early              | Broken delegation or DNSSEC issue   |
| TTL very high (86400)               | `dig +noall +answer` shows high value | Old data persisting too long        |
| Authoritative NS missing record     | `dig @ns1.example.com` → no result    | Zone file not updated               |
| Site resolves but not loads         | `curl -I` or `ping` fails             | App down / Firewall / SSL misconfig |

---

## 🧠 Example Outputs for Reference

✅ **Good case:**

```bash
$ dig example.com @8.8.8.8 +short
93.184.216.34
```

⚠️ **Propagation case:**

```bash
$ dig example.com @8.8.8.8 +short
203.0.113.42
$ dig example.com @1.1.1.1 +short
198.51.100.17
```

❌ **NXDOMAIN:**

```bash
$ dig example.com
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 9999
```

---

## 🔧 Summary — Key DNS Troubleshooting Mindset

1. **Always start broad (public resolvers)** → then go deep (authoritative).
2. **Compare** multiple sources → detect cache or propagation.
3. **Trace** when you suspect delegation issues.
4. **Check TTL** before panicking — many “errors” are just waiting periods.
5. **Separate DNS vs App** — confirm DNS works before blaming the service.

---

Would you like me to build the **same upgraded table** (step-by-step + fallback + why) for **Kubernetes troubleshooting** next — in exactly this structured, lab-style format? It would cover pods, services, deployments, logs, and networking.
