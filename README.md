# T-Pot Honeypot Study (AWS)

I wanted to see what actually hits a server when you put it on the internet, so I ran a T-Pot honeypot on AWS for about two days (Nov 17–19, 2026) and looked at the data.

It got hit **324,000+ times** in that window. Most of it was automated scanning, but there was a lot more going on than I expected.

## Setup

- **Platform:** [T-Pot](https://github.com/telekom-security/tpotce) (Hive install). It bundles ~20 honeypots plus Elasticsearch/Kibana and Suricata.
- **Server:** AWS EC2 t3.xlarge (16 GB RAM, 256 GB disk), Ubuntu 26.04, us-east-2
- **Firewall:** SSH (64295) and the web UI (64297) locked to my IP. Every other TCP/UDP port was open to the internet.

The honeypots run in Docker containers, so attackers only ever interact with fake services, never the actual host.

A few things went wrong during setup, which I'm leaving here in case they help someone:
- The current T-Pot release needs Ubuntu 26.04. I started on 24.04 and the installer refused to run, so I had to rebuild the instance.
- After the install, SSH moves to port 64295. I rebooted before updating the security group and locked myself out for a bit.
- When I stopped/started the instance, AWS gave it a new public IP. That took me a minute to figure out.

## What I found

### Most of the traffic was aimed at phone systems

| Honeypot | Emulates | Hits |
|---|---|---|
| Sentrypeer | VoIP / SIP | ~230k |
| Honeytrap | any port | ~51k |
| H0neytr4p | web server | ~16k |
| Tanner | web app | ~15k |
| Dionaea | SMB, etc. | ~8k |
| Cowrie | SSH / Telnet | ~2k |

I didn't expect VoIP to be the top target. Suricata flagged **SIPVicious** and **Sipsak** scans, which are tools people use to find exposed phone systems. The usual goal is toll fraud: break into a PBX and make expensive international calls on someone else's bill.

Top ports overall were 5060 (SIP), 443/80 (web), 1433 (MS SQL), and 445 (SMB).

### One IP did almost half of it

| IP | Hits | Provider |
|---|---|---|
| 188.165.209.25 | 135,888 | OVH |
| 172.110.223.9 | 52,983 | Husam A. H. Hijazi |
| 161.97.85.154 | 30,424 | Contabo |
| 172.110.223.110 | 23,076 | Husam A. H. Hijazi |
| 68.183.53.135 | 18,035 | DigitalOcean |

`188.165.209.25` alone accounts for about 42% of everything. Three of the top IPs are in the same `172.110.223.x` range, which looks like one operator spreading the load across addresses.

Almost all of the heavy hitters were cheap VPS/cloud providers, not home connections. The country breakdown (France, Philippines, US, UK, Canada) mostly reflects where those servers are hosted, not where the people behind them are.

Most of the source IPs were already tagged as "bad reputation" or "known scanner" in T-Pot's threat intel lists.

### Passwords

The most-tried usernames were `sa` (SQL Server admin), `root`, `admin`, and `postgres`. The most-tried passwords were a blank password, `123456`, `root`, `admin`, `password`, and `master`.

What stood out was how many were **device factory defaults**:
- `7ujMko0admin`: from the Mirai botnet's credential list
- `chtngpon`, `Zte521`, `adminHW`, `CTLSupport12`: router/modem defaults
- `Hi3518`: IP camera default

So a lot of this is bots looking for routers and cameras that were never reconfigured.

### Old exploits are still being tried

Suricata picked up **DoublePulsar** installation attempts. That's the NSA backdoor that leaked in 2017 and was used in WannaCry. Most of the CVEs it matched were from 1999–2012. People keep scanning for old bugs because unpatched systems are still out there.

There were also requests for `.env` files, which usually means someone fishing for leaked API keys or database passwords.

### 22 "successful" logins, but nobody did anything

Cowrie (the fake SSH/Telnet server) intentionally accepts some logins so you can watch what happens next. 22 sessions got in (16 Telnet, 6 SSH), but:
- No commands were run
- No files were downloaded
- Dionaea didn't capture any malware either

My read is that these bots were just **checking which credentials work**, probably to sell the list or come back later with something else. A longer run would probably catch actual payloads.

A few of the "logins" weren't really logins. They were HTTP requests sent to the Telnet port that Cowrie logged as usernames (`User-Agent: ...`, `Accept-Encoding: gzip`). A couple of others came from research scanners like Shadowserver.

The cleaned-up list is in [`successful_logins_clean.csv`](successful_logins_clean.csv).

## MITRE ATT&CK

| Technique | ID | Seen as |
|---|---|---|
| Active Scanning | T1595 | port scans, Nmap, SIP scanners |
| Password Guessing | T1110.001 | SSH/Telnet/SQL brute force |
| Default Accounts | T1078.001 | router/IoT default creds |
| Exploit Public-Facing App | T1190 | web exploit attempts, `.env` requests |
| Exploitation of Remote Services | T1210 | SMB / DoublePulsar attempts |

## Takeaways

Nothing here was sophisticated. It was almost all automated scanning and password guessing, and that's kind of the point. If something is exposed with a default password, it gets found fast. Basic stuff like changing default creds, not exposing Telnet/SMB/SIP/SQL to the internet, and patching old systems would stop the vast majority of what I saw.

## Files

- `successful_logins_clean.csv`: the 22 Cowrie logins (IP, country, provider, protocol, username, password)
- `screenshots/`: Kibana dashboards from the run

## Notes

- Ran in a separate AWS account, not connected to any personal or work network.
- No malware was downloaded or executed.
- The instance has been terminated.
