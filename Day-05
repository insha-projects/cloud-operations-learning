# Day 5 — Firewall Rules and External Reachability

Dates: October 19–20, 2026  
Type: Guided personal lab

## 1. Objective

Configure Ubuntu’s UFW firewall, preserve SSH access, and compare local application availability with access from Windows.

Demonstrate that a service can remain active and listening while incoming requests are blocked by a firewall.

## 2. Environment

- Windows 11 host running VirtualBox
- Ubuntu Server VM: Ubuntu-Cloud-Lab
- Nginx serving my personal lab webpage on TCP port 8080
- UFW managing Ubuntu’s firewall rules
- SSH access through VirtualBox port forwarding
- No paid cloud resources used

Port forwarding used:

| Purpose | Windows address and port | Ubuntu destination port |
|---|---|---|
| SSH | 127.0.0.1:2222 | 22 |
| Webpage | 127.0.0.1:8080 | 8080 |

The Windows-side forwarding entries were bound to loopback, limiting those entry points to the Windows host.

## 3. What I Did

### Checked the starting state

Inside Ubuntu:

```bash
systemctl is-active nginx
```

Result: `active`.

```bash
curl --noproxy '*' --connect-timeout 5 http://127.0.0.1:8080/
```

Response:

```text
Insha personal Linux lab - Day 1
```

```bash
sudo ufw status verbose
```

Result: `Status: inactive`.

This established that Nginx responded locally before enabling UFW.

### Prepared SSH access

Inside Ubuntu:

```bash
sudo ss -ltnp 'sport = :22'
```

The output showed listeners on port 22 for IPv4 and IPv6, associated with sshd and systemd.

```bash
sudo ufw allow 22/tcp
```

UFW reported:

```text
Rules updated
Rules updated (v6)
```

I allowed Ubuntu’s port 22 rather than Windows port 2222 because VirtualBox forwards the connection to port 22 inside the VM.

### Established a Windows web-request baseline

I configured a VirtualBox NAT port-forwarding rule:

| Field | Value |
|---|---|
| Name | day5-web |
| Protocol | TCP |
| Host IP | 127.0.0.1 |
| Host Port | 8080 |
| Guest IP | Blank |
| Guest Port | 8080 |

In Windows PowerShell:

```powershell
curl.exe --noproxy "*" --connect-timeout 5 --max-time 10 http://127.0.0.1:8080/
```

The request returned my personal lab webpage.

This tested the path from Windows through VirtualBox to Ubuntu, rather than a request originating inside Ubuntu.

### Configured and enabled UFW

Inside Ubuntu:

```bash
sudo ufw default deny incoming
```

This set the default policy to deny incoming traffic unless allowed.

```bash
sudo ufw default allow outgoing
```

This permitted outgoing traffic by default.

```bash
sudo ufw show added
```

I checked that the saved SSH allow rule was present before enabling the firewall.

```bash
sudo ufw enable
```

I confirmed the prompt warning that the operation could disrupt SSH connections.

```bash
sudo ufw status verbose
```

The output showed:

- Status: active
- Default incoming policy: deny
- Default outgoing policy: allow
- TCP port 22 allowed for IPv4 and IPv6
- No allow rule for TCP port 8080 at this stage

### Repeated the Windows request

In Windows PowerShell:

```powershell
curl.exe --noproxy "*" --connect-timeout 5 --max-time 10 http://127.0.0.1:8080/
```

The request timed out with curl error 28 after approximately ten seconds, with zero bytes received.

The same Windows request had succeeded before UFW was enabled.

### Checked whether Nginx still worked locally

Inside Ubuntu:

```bash
systemctl is-active nginx
```

Result: `active`.

```bash
sudo ss -ltnp 'sport = :8080'
```

The output showed Nginx listening on `0.0.0.0:8080`.

```bash
curl --noproxy '*' --connect-timeout 5 http://127.0.0.1:8080/
```

My personal lab webpage still responded.

This showed that the service and listener remained available locally while Windows access failed.

### Allowed incoming web requests

Inside Ubuntu:

```bash
sudo ufw allow 8080/tcp
```

UFW reported that rules were added for IPv4 and IPv6.

```bash
sudo ufw status verbose
```

The firewall remained active, with ports 22/tcp and 8080/tcp allowed.

In Windows PowerShell:

```powershell
curl.exe --noproxy "*" --connect-timeout 5 --max-time 10 http://127.0.0.1:8080/
```

Response:

```text
Insha personal Linux lab - Day 1
```

Windows access returned after adding the firewall rule. I did not restart or reconfigure Nginx.

### Verified a new SSH connection

I kept my existing SSH session open and opened another Windows Terminal tab.

From Windows:

```powershell
ssh -p 2222 ubuntu_vm@127.0.0.1
```

The new connection succeeded.

Inside the new Ubuntu session:

```bash
sudo ufw status verbose
```

UFW was active with SSH and web access allowed.

### Final state

I left UFW enabled with:

- Incoming traffic denied by default
- Outgoing traffic allowed by default
- TCP port 22 allowed
- TCP port 8080 allowed

Nginx remained running, and the VirtualBox web-forwarding rule remained configured.

## 4. What I Verified

| Check | Actual result | What it established |
|---|---|---|
| Initial Nginx status | active | The service was active |
| Initial local request | Lab webpage returned | Nginx responded inside Ubuntu |
| Initial UFW status | inactive | UFW was not enforcing its rules |
| SSH listener check | Port 22 listeners present | SSH had listening sockets |
| Windows request before UFW | Lab webpage returned | The forwarded Windows-to-Ubuntu path worked |
| UFW enabled with SSH allowed | Active; incoming denied by default | The firewall configuration was applied |
| Windows request without an 8080 allow rule | Timeout; curl error 28 | The request did not complete within the time limit |
| Local checks while Windows access failed | Service active, listener present, webpage returned | The application remained available locally |
| Windows request after allowing 8080 | Lab webpage returned | Web access returned after the firewall change |
| New SSH connection | Successful | A fresh SSH connection worked with UFW active |

The before-and-after tests support the conclusion that UFW filtering caused the Windows access failure in this controlled lab.

A timeout by itself would not establish that a firewall was the cause.

### Evidence

![Nginx response and UFW status before the lab](screenshots/01-starting-state.png)

![SSH listener and prepared firewall rule](screenshots/02-ssh-rule-prepared.png)

![Successful Windows request before enabling UFW](screenshots/03-windows-web-baseline.png)

![UFW configuration and Windows timeout after enabling it](screenshots/04-firewall-enabled-and-windows-timeout.png)

![Local Nginx checks while UFW was active](screenshots/05-local-service-with-firewall-active.png)

![Web allow rule and successful Windows retest](screenshots/06-web-rule-and-windows-retest.png)

![UFW status in the newly established SSH session](screenshots/07-new-ssh-connection.png)

## 5. Problems and Fixes

### Windows could not reach the webpage after enabling UFW

This was an intentional lab condition.

The firewall denied incoming traffic by default, and port 8080 had no allow rule.

I checked the service, listener, and local webpage response before changing anything else. All three checks succeeded.

I added an allow rule for TCP port 8080 and repeated the Windows request. The webpage responded again without restarting Nginx.

### Different results inside Ubuntu and from Windows

These requests followed different paths.

The request inside Ubuntu used loopback, which UFW’s standard rules permit.

The Windows request entered Ubuntu through VirtualBox’s NAT port forwarding and was subject to incoming firewall filtering.

A successful loopback request therefore did not prove that Windows could reach the service.

## 6. What I Learned

### My Dunkin explanation

Nginx is like the Dunkin drive-through operation. Its processes do the work, and its listener is the operating order point at a numbered window.

The firewall is like a gate controlling customer access to that window.

During this lab, the store was still operating and the order window was staffed. Someone inside could get coffee, but customers outside could not reach the window because the gate did not permit access.

Allowing TCP port 8080 opened the relevant route through the gate. The staff did not need to restart because the store had been working all along.

Windows itself was not banned. The incoming request to the web service’s destination port was not permitted.

### Key distinctions

| Concept | What I learned |
|---|---|
| Service active | The service is running; this does not guarantee external access |
| Listener present | A program is accepting connections on an address and port |
| Firewall allow rule | Permits matching traffic; it does not start a listener |
| Port forwarding | Passes connections from a host port to a guest destination |
| Local request | Tests the application through the VM’s loopback path |
| Windows request | Tests access through VirtualBox forwarding and Ubuntu’s incoming firewall |
| Timeout | The operation did not complete in time; its cause requires investigation |

### Teach-it-back questions

**1. Can Nginx be running while Windows cannot reach it?**

Yes. A firewall can block incoming traffic even when the service and listener are working.

**2. Why did I allow port 22 instead of 2222 inside Ubuntu?**

Windows port 2222 forwards to Ubuntu port 22. UFW filters the connection at the Ubuntu destination.

**3. Why did the local webpage still work?**

It used Ubuntu’s loopback path, which was permitted by the standard UFW rules.

**4. Did allowing port 8080 start or restart Nginx?**

No. It permitted incoming traffic to an already-running listener.

**5. Why test a new SSH connection?**

An existing connection can remain usable without proving that a fresh connection will be accepted.

## 7. Scope

This was a guided personal lab on a local Ubuntu VM.

I practiced basic UFW configuration, preserving SSH access, comparing local and Windows requests, and verifying the effect of a firewall rule.

The requests originated on my Windows host. I did not test access from the public internet.

Safe to say now:
“Configured and tested Ubuntu UFW rules in a guided personal lab, preserving SSH access and restoring web access after comparing service, listener, and client checks.”

Do not claim yet:
Production firewall administration, cloud security-group experience, or comprehensive server hardening.
