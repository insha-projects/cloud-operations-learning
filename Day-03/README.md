# Day 3 — IP Addresses, Ports and Connection Troubleshooting

**Lab date:** October 15, 2026  
**Scope:** Guided personal lab; no production systems or cloud deployment

## 1. Objective

Understand the difference between a running service, its network
listeners and the response it provides.

Practice distinguishing:
- A working webpage.
- A connection attempt to an unused port.
- A missing resource on a working web service.
- A stopped service with no listener.

## 2. Environment

| Component | Configuration |
|---|---|
| Host operating system | Windows 11 |
| Virtualization software | VirtualBox 7.2.20 |
| Guest operating system | Ubuntu Server 26.04.1 LTS |
| Administration | SSH from Windows Terminal |
| Web server | Nginx |
| Custom page | `/var/www/cloud-lab/index.html` |
| Custom-page port | TCP 8080 |
| Tools | `ip`, `ss`, `systemctl`, `curl` |

All Linux commands ran inside Ubuntu through the SSH connection.

## 3. What I Did

### A. Checked the Working Page

```bash
curl --noproxy '*' http://127.0.0.1:8080/
```

This requested the custom page directly, without using a configured proxy.

**Observed response:**

```text
Insha personal Linux lab - Day 1
```

Inside the SSH session, `127.0.0.1` referred to Ubuntu because
Ubuntu was executing curl.

### B. Inspected Network Interfaces and Addresses

```bash
ip -br address
```

This displayed a brief list of network interfaces and their addresses.

**Observed:**
- `lo` had the loopback address `127.0.0.1`.
- `enp0s3` was the VM's virtual network interface and had an IPv4 address.
- The loopback interface showed `UNKNOWN`, which did not indicate
  a failure in this lab.

A network interface is a connection point through which a computer
sends and receives network traffic.

### C. Inspected Routes

```bash
ip route
```

This displayed the routing table.

I identified a default route through the VirtualBox NAT gateway.

A route tells the computer where to send traffic. The default route
is used when no more specific route matches the destination.

The route listing alone did not prove internet access.

### D. Inspected Listening TCP Ports

```bash
sudo ss -ltnp
```

| Option | Meaning |
|---|---|
| `-l` | Show listeners |
| `-t` | Show TCP sockets |
| `-n` | Show numeric addresses and ports |
| `-p` | Show associated process information where available |

**Observed:**
- Nginx listened on ports 80 and 8080.
- SSH listened on port 22.
- The custom Nginx listener appeared as `0.0.0.0:8080`.

`0.0.0.0:8080` indicated listening across the VM's IPv4 interfaces.
It did not prove the service was accessible from the public internet.

### E. Compared a Working Port With an Unused Port

I requested the working page with response headers:

```bash
curl --noproxy '*' -i http://127.0.0.1:8080/
```

The `-i` option included HTTP response headers.

**Observed:**
- HTTP status: `200 OK`
- Body: my custom page text

I checked for a listener on port 8081:

```bash
sudo ss -ltnp 'sport = :8081'
```

**Observed result:** Headings appeared, but no listener rows.

I attempted a connection to that port:

```bash
curl --noproxy '*' --connect-timeout 3 --max-time 5 http://127.0.0.1:8081/
```

- `--connect-timeout 3` limited the connection phase.
- `--max-time 5` limited the entire operation.

**Observed result:** Curl could not connect.

Immediately afterward, I checked the exit code:

```bash
echo $?
```

**Observed result:** `7`, indicating a connection failure.

The same IP address worked on 8080 but failed on 8081 because
Nginx was listening on 8080 and no listener was present on 8081.

### F. Requested a Missing Resource

```bash
curl --noproxy '*' -i http://127.0.0.1:8080/missing-page
```

**Observed result:** `HTTP/1.1 404 Not Found`, followed by
an HTML error page.

Immediately afterward:

```bash
echo $?
```

**Observed result:** `0`.

Nginx accepted the connection and responded, but it could not find
the requested resource.

By default, curl does not treat HTTP 404 as a command failure.
The transfer completed even though the requested page was missing.

This showed why an exit code of zero alone does not prove that
the expected webpage was returned.

### G. Stopped Nginx and Observed the Listener Disappear

I deliberately stopped the local web service:

```bash
sudo systemctl stop nginx
```

I checked its state:

```bash
systemctl is-active nginx
```

**Observed result:** `inactive`.

This was an intentional stop, unlike Day 2's failed startup.

I checked port 8080:

```bash
sudo ss -ltnp 'sport = :8080'
```

**Observed result:** No listener rows.

I requested the page:

```bash
curl --noproxy '*' --connect-timeout 3 --max-time 5 http://127.0.0.1:8080/
```

**Observed result:** Curl reported error 7 and could not connect.

There was no HTTP response because the stopped service could
not accept the connection.

### H. Started Nginx and Verified Recovery

```bash
sudo systemctl start nginx
```

I checked the listener:

```bash
sudo ss -ltnp 'sport = :8080'
```

**Observed result:** A `LISTEN` row associated with Nginx returned.

I requested the page:

```bash
curl --noproxy '*' http://127.0.0.1:8080/
```

**Observed response:**

```text
Insha personal Linux lab - Day 1
```

Starting Nginx restored both the listener and the page response.

## 4. What I Verified

| Scenario | Listener | Request result | Meaning |
|---|---|---|---|
| Working service on 8080 | Present | HTTP 200 and custom page | Service responded with expected content |
| Unused port 8081 | Absent | Connection failure; curl exit 7 | No accepting service at that endpoint |
| Missing page on 8080 | Present | HTTP 404; curl exit 0 | Service responded, but resource was missing |
| Nginx deliberately stopped | Absent on 8080 | Connection failure; curl error 7 | Stopped service could not accept requests |
| Nginx started again | Present on 8080 | Custom page returned | Listener and page response recovered |

## 5. Problems and Fixes

The connection failures and HTTP 404 were deliberate tests,
not unexpected faults.

For the stopped-service test, I restored operation with
`sudo systemctl start nginx`, then checked the listener and
requested the page.

I initially found it difficult to distinguish services, processes,
listeners and ports. The Dunkin' drive-through analogy helped
me understand how they relate.

Screenshots were retained locally. No screenshot uploads are
required for this write-up.

## 6. What I Learned

### My Dunkin' Drive-Through Example

Think of Ubuntu as one Dunkin' location.

| Computing term | Dunkin' example |
|---|---|
| IP address | The location's address |
| Port | A numbered service window |
| Nginx service | The drive-through operation |
| Processes | The manager and workers carrying out the operation |
| Listener | An operating order point ready to accept customers |
| curl client | A customer placing an order |
| Request | The customer's order |
| Resource | The item requested |
| Response | The item or a reply explaining the outcome |
| Firewall | A gate controlling access to the windows |

This analogy helps explain the relationships. A real listener
is a software networking endpoint, not a physical opening.

### Service Versus Process

The service is the managed operation. Processes are running
instances of programs doing its work.

In the Dunkin' example, the drive-through operation is the service,
and the manager and workers represent processes.

One service can involve several processes. Not every process
belongs to a background service: curl runs as a process, makes
its request and exits.

### Service Versus Listener

A service can create and use listeners to accept network connections.

The service listens on a port. The port itself does not listen.

One service can have multiple listeners. My Nginx service accepted
requests on ports 80 and 8080.

A service can also run without a network listener. For example,
a local backup service can copy files without accepting incoming
network requests.

In the Dunkin' example, staff can perform work inside the store
even when no drive-through order point is operating.

### Why Port 8081 Failed

The restaurant operation could be running at window 8080,
while window 8081 had nobody accepting orders.

Likewise, Nginx could be running on port 8080 while a connection
to port 8081 failed.

In this lab, I checked with ss that 8081 had no listener.
In a different situation, a connection failure could also involve
a firewall or network problem.

### Why HTTP 404 Is Different

A 404 is like reaching an operating Dunkin' window, placing
an order and being told the requested item is unavailable.

The listener accepted the connection and the service responded.
The resource was missing.

A connection failure is different: no HTTP response was received.

### HTTP Status Versus Curl Exit Code

The HTTP status describes the web server's response:
- `200 OK`: the request succeeded.
- `404 Not Found`: the requested resource was not found.

Curl's exit code describes the command's transfer outcome:
- `0`: curl completed successfully under the options used.
- `7`: curl could not connect.

A 404 response and curl exit code 0 can occur together.
I must inspect the HTTP status and content, not just the exit code.

### My Troubleshooting Sequence

1. Use systemctl to check whether the service is active.
2. Use ss to check for a listener on the intended address and port.
3. Use curl to request the resource and inspect the response.

In my own words:

“First, I check whether the service itself is running.
Then I check whether it has a listener on the intended port.
Then I request the page to check whether it provides the resource.”

In the Dunkin' example:

“Is the drive-through operation running?
Is the order point accepting customers?
Can I actually place an order and receive the expected item?”

These checks answer different questions. An active service alone
does not prove the expected webpage works.

## 7. Scope

This was a guided personal networking lab on a local Ubuntu VM.

**Safe to say now:** Used ip, ss, systemctl and curl to inspect
local networking and distinguish successful HTTP responses,
missing resources and connection failures.

**Do not claim yet:** Production network troubleshooting,
VPC administration, enterprise firewall management or
professional cloud-support experience.
