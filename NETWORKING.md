
---

# 🌐 NETWORK TROUBLESHOOTING PLAYBOOK — FULL PROFESSIONAL VERSION

*(For ARM / NOC / DevOps engineers — with When/Why/Next)*

|  🧭 Step | 🕒 When to Use                             | 🎯 Purpose / Question          | 💻 Command                                 | 🧾 Expected Output                            | 🔄 If Unexpected → Next Action           | 🧠 Why / Explanation                               |
| :------: | :----------------------------------------- | :----------------------------- | :----------------------------------------- | :-------------------------------------------- | :--------------------------------------- | :------------------------------------------------- |
|  **1️⃣** | Immediately when host can’t reach anything | Check if interface has an IP   | `ip addr show` OR `ifconfig`               | ✅ Interface shows IP (e.g. `192.168.1.10/24`) | ❌ No IP → Step 2                         | Without IP, system can’t send/receive packets.     |
|  **2️⃣** | When Step 1 shows no IP or “down”          | Request IP from DHCP           | `sudo dhclient -v`                         | ✅ Gets IP lease                               | ❌ Fails → Step 3                         | Confirms DHCP works and host can join LAN.         |
|  **3️⃣** | After getting IP, but still no Internet    | Check default route/gateway    | `ip route show`                            | ✅ Default via `192.168.1.1`                   | ❌ None → add route / check DHCP          | Gateway defines next hop to Internet.              |
|  **4️⃣** | When IP and gateway exist                  | Test local LAN connectivity    | `ping -c 4 192.168.1.1`                    | ✅ Replies OK                                  | ❌ Timeout → Step 5                       | Verifies LAN path and cable/switch function.       |
|  **5️⃣** | When LAN reachable but no Internet         | Test external reachability     | `ping -c 4 8.8.8.8`                        | ✅ Replies OK                                  | ❌ Timeout → Step 6                       | Checks Internet path beyond LAN.                   |
|  **6️⃣** | When `ping 8.8.8.8` fails                  | Trace route to external host   | `traceroute 8.8.8.8`                       | ✅ Ends at 8.8.8.8                             | ⚠️ Stops mid-route → Step 7              | Finds where packet dies (ISP/firewall).            |
|  **7️⃣** | After traceroute issue found               | Check route table details      | `ip route get 8.8.8.8`                     | ✅ via correct gateway                         | ❌ Wrong route → fix config               | Ensures packets take correct exit interface.       |
|  **8️⃣** | If ping to IP works but website fails      | Test DNS resolution            | `ping -c 4 google.com`                     | ✅ Resolves IP and pings                       | ⚠️ “unknown host” → Step 9               | Determines if issue is DNS-related.                |
|  **9️⃣** | When name can’t resolve                    | Manually test DNS              | `dig google.com +short`                    | ✅ Returns IP                                  | ❌ NXDOMAIN → check resolvers             | Confirms DNS functionality independent of app.     |
|  **🔟**  | When DNS works but websites still fail     | Test HTTP(S) connection        | `curl -I https://google.com`               | ✅ `HTTP/1.1 200 OK`                           | ❌ Timeout → Step 11                      | Checks application-level connectivity (Layer 7).   |
| **11️⃣** | When ping works but ports closed           | Test TCP/UDP port connectivity | `nc -vz google.com 443`                    | ✅ “succeeded”                                 | ❌ Refused → Step 12                      | Confirms specific port (80/443/etc.) reachability. |
| **12️⃣** | When local service not reachable           | List listening ports           | `sudo netstat -tulnp` OR `ss -ltnp`        | ✅ Port bound (e.g. 80, 443)                   | ❌ Missing → app not running              | Ensures local service is actually listening.       |
| **13️⃣** | When service is up but blocked             | Check firewall rules           | `sudo ufw status` OR `sudo iptables -L -n` | ✅ Ports allowed                               | ⚠️ Drop rules → adjust                   | Verifies local firewall not filtering.             |
| **14️⃣** | When intermittent latency or packet loss   | Continuous path test           | `mtr -rw 8.8.8.8`                          | ✅ Stable latency                              | ⚠️ Packet loss → network issue           | Combines ping + traceroute for live diagnostics.   |
| **15️⃣** | When site resolves but doesn’t load fully  | Full HTTP(S) test              | `curl -v https://api.example.com`          | ✅ 200 OK                                      | ❌ SSL error / timeout → Step 16          | Shows transport & SSL negotiation details.         |
| **16️⃣** | When DNS OK but routing fails              | Check both DNS + route         | `dig +trace google.com` + `ping <IP>`      | ✅ IP resolves & pings                         | ❌ Ping fails → remote firewall           | Differentiates local vs remote failure.            |
| **17️⃣** | When VPN/tunnel causes timeout             | Test MTU size                  | `ping -M do -s 1472 google.com`            | ✅ Replies                                     | ⚠️ “Frag needed” → lower MTU (e.g. 1400) | Diagnoses fragmentation issues inside tunnels.     |
| **18️⃣** | When external connection behaves oddly     | Verify public IP               | `curl ifconfig.me`                         | ✅ Shows public IP                             | ⚠️ No response → NAT / proxy issue       | Checks if outbound path translated correctly.      |
| **19️⃣** | When remote users can’t reach your server  | Test inbound reachability      | From external host: `ping your_public_IP`  | ✅ Replies                                     | ❌ Timeout → inbound firewall issue       | Confirms inbound route and firewall health.        |
| **20️⃣** | After fix implemented                      | Confirm full recovery          | Repeat `ping 8.8.8.8`, `dig`, `curl`       | ✅ All OK                                      | ✅ Incident closed                        | Ensures connectivity restored end-to-end.          |

---

## 🧩 WHEN TO USE EACH COMMAND — SHORT SUMMARY

| Tool                   | Use When                    | Purpose                         |
| ---------------------- | --------------------------- | ------------------------------- |
| `ip addr show`         | No connectivity at all      | Verify interface status & IP    |
| `dhclient -v`          | No IP assigned              | Request DHCP address            |
| `ip route show`        | Internet unreachable        | Check gateway & default route   |
| `ping <gateway>`       | Local LAN test              | Confirms link to gateway/router |
| `ping 8.8.8.8`         | Internet check              | Confirms global reachability    |
| `traceroute 8.8.8.8`   | Partial connectivity        | Locate broken hop               |
| `dig domain.com`       | Website fails but IP works  | Test DNS                        |
| `curl -I https://site` | DNS works but site fails    | Test HTTP(S) response           |
| `nc -vz host port`     | Port suspected blocked      | Check TCP port                  |
| `netstat / ss`         | Local service unreachable   | Verify process bound to port    |
| `ufw / iptables`       | Block suspected             | Check firewall rules            |
| `mtr`                  | High latency / intermittent | Continuous path visibility      |
| `ping -M do -s`        | VPN drops packets           | Test MTU limit                  |
| `curl ifconfig.me`     | Outbound IP check           | Verify NAT/public IP            |
| `ping from external`   | Remote users report outage  | Confirm inbound traffic         |
| `dig +trace`           | DNS delegation suspected    | Follow entire resolution path   |

---

## 🔍 Example Outputs (Good vs Bad)

✅ **Good Network State**

```bash
$ ping -c 2 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=22.3 ms
```

⚠️ **Local Network Issue**

```bash
$ ping -c 2 192.168.1.1
Destination Host Unreachable
```

❌ **Firewall or ISP Drop**

```bash
$ traceroute 8.8.8.8
1  192.168.1.1 (192.168.1.1)  0.512 ms
2  * * *  (timeout)
```

---

## 🧠 Analytical Thinking Tips

| Pattern                          | What It Means            | Action                       |
| -------------------------------- | ------------------------ | ---------------------------- |
| Ping fails to gateway            | Local NIC / switch issue | Check cable or VLAN          |
| Ping works to IP, not to domain  | DNS issue                | Run `dig`                    |
| Ping works to domain, curl fails | App layer / firewall     | Run `nc` and `curl -v`       |
| High latency / jitter in `mtr`   | Route congestion         | Escalate to network provider |
| `Frag needed` in ping            | MTU mismatch             | Adjust MTU in VPN config     |

---

## 🧭 Network Diagnostic Flow (Cheat Sheet)

```text
[Local Host]
  ↓ ip addr / ifconfig
[LAN Connectivity]
  ↓ ping gateway
[Internet Layer]
  ↓ ping 8.8.8.8 → traceroute
[DNS Layer]
  ↓ dig domain.com
[App Layer]
  ↓ curl / nc / mtr
[Confirm Fix]
  ↓ repeat ping / curl / dig
✅ Done
```

---

## 🚀 Final Notes

* Always **document** each command and output in your incident report.
* Move **outward logically** (local → gateway → Internet → DNS → service).
* Don’t skip layers — 80% of “Internet issues” are actually **DNS or firewall** problems.
* Combine this playbook with the **DNS troubleshooting table** for complete coverage.

---

Would you like me to now build the same **“step-by-step + when to use” table for Kubernetes networking problems** (e.g. Pods can’t reach Services, DNS inside cluster fails, Ingress not working)?
That’s the next logical layer after this network playbook.
