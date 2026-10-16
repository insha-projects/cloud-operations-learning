# Day 4 — DNS Queries and Troubleshooting

Date: October 16, 2026  
Type: Guided personal lab

## 1. Objective

Query DNS servers, inspect returned addresses, and distinguish a DNS lookup problem from an application problem.

Create an intentionally incorrect local DNS record, correct it, and verify the DNS answer separately from the webpage response.

## 2. Environment

- Windows 11 host with VirtualBox
- Ubuntu Server VM
- Nginx serving my personal lab webpage on port 8080
- dig version: 9.20.24-1ubuntu0.3-Ubuntu
- dnsmasq version: 2.92
- Temporary lab DNS listener: 127.0.0.1:1053
- Lab name: cloud-lab.test
- Configuration file: ~/day-04-dns/dnsmasq.conf

The temporary DNS server did not replace Ubuntu’s normal DNS settings.

## 3. What I Did

### Checked the existing application

```bash
systemctl is-active nginx
```

Result: `active`.

```bash
curl --noproxy '*' http://127.0.0.1:8080/
```

Response:

```text
Insha personal Linux lab - Day 1
```

This confirmed that the local application responded directly by IP address. It did not test DNS.

### Queried a public name

```bash
dig ubuntu.com A
```

I requested A records, which map a name to IPv4 addresses.

The response showed:

- Status: NOERROR
- Seven A-record answers
- Answering resolver: 127.0.0.53 on port 53

### Queried a deliberately nonexistent name

```bash
dig cloud-lab.invalid A
```

The response showed:

- Status: NXDOMAIN
- Zero answer records

The resolver responded that the name did not exist. This was different from receiving no answer before a timeout.

### Compared DNS lookup and application behavior

```bash
curl --noproxy '*' --connect-timeout 5 http://cloud-lab.invalid:8080/
```

Actual result:

```text
curl: (28) Resolving timed out after 5002 milliseconds
```

```bash
echo $?
```

Exit code: `28`.

The expected result in the guide was a hostname-resolution error with exit code 6. My actual result was a timeout during resolution, so I documented that instead.

I checked Ubuntu’s system name-resolution path:

```bash
getent ahostsv4 cloud-lab.invalid
```

It returned immediately without printing an address.

```bash
echo $?
```

Exit code: `2`.

I then requested the application directly:

```bash
curl --noproxy '*' --connect-timeout 5 http://127.0.0.1:8080/
```

My lab webpage still responded. The name-based request had failed during resolution, while the direct-IP request worked.

### Selected a specific DNS server

```bash
dig @1.1.1.1 ubuntu.com A +time=3 +tries=1
```

The response showed:

- Status: NOERROR
- Seven A-record answers
- Answering server: 1.1.1.1 on port 53
- Query time: 295 milliseconds

This selected a server for one query. It did not change Ubuntu’s saved DNS settings.

### Inspected Ubuntu’s resolver configuration

```bash
resolvectl status
```

I inspected the local resolver mode and the DNS servers configured for the VM’s network interface.

Private network addresses are omitted from these public notes.

### Installed the lab DNS tool

```bash
sudo apt update
```

This refreshed the available package information.

```bash
sudo apt install dnsmasq-base
```

This installed the program used for the temporary DNS lab.

```bash
dnsmasq --version
```

Installed version: `2.92`.

### Created an intentionally incorrect record

```bash
mkdir -p ~/day-04-dns
```

```bash
nano ~/day-04-dns/dnsmasq.conf
```

Initial configuration:

```text
port=1053
listen-address=127.0.0.1
bind-interfaces
no-resolv
no-hosts
address=/cloud-lab.test/192.0.2.10
```

The lab listener was restricted to the VM’s loopback address. It used port 1053 and had no upstream DNS servers configured.

The address 192.0.2.10 was the deliberately incorrect destination.

I validated the configuration:

```bash
dnsmasq --test --conf-file="$HOME/day-04-dns/dnsmasq.conf"
```

Result:

```text
dnsmasq: syntax check OK.
```

A successful syntax check did not establish that the address was correct.

### Started and queried the temporary DNS server

```bash
sudo dnsmasq --keep-in-foreground --conf-file="$HOME/day-04-dns/dnsmasq.conf" --log-queries --log-facility=-
```

I left that terminal running and used a second SSH session for queries.

```bash
dig @127.0.0.1 -p 1053 cloud-lab.test A
```

The response showed:

- Status: NOERROR
- One A-record answer
- Returned address: 192.0.2.10
- Answering server: 127.0.0.1 on port 1053

The lookup succeeded, but the configured address was wrong.

### Corrected the address record

After stopping the temporary DNS process, I edited the configuration:

```bash
nano ~/day-04-dns/dnsmasq.conf
```

I changed the final line to:

```text
address=/cloud-lab.test/127.0.0.1
```

I checked the configuration again:

```bash
dnsmasq --test --conf-file="$HOME/day-04-dns/dnsmasq.conf"
```

I started the lab DNS process again:

```bash
sudo dnsmasq --keep-in-foreground --conf-file="$HOME/day-04-dns/dnsmasq.conf" --log-queries --log-facility=-
```

Then I repeated the query from the second terminal:

```bash
dig @127.0.0.1 -p 1053 cloud-lab.test A
```

The response now showed:

- Status: NOERROR
- One A-record answer
- Corrected address: 127.0.0.1

### Tested the application separately

```bash
curl --noproxy '*' --connect-timeout 5 --resolve cloud-lab.test:8080:127.0.0.1 http://cloud-lab.test:8080/
```

Response:

```text
Insha personal Linux lab - Day 1
```

```bash
echo $?
```

Exit code: `0`.

The --resolve option supplied the address directly for this curl command. It bypassed DNS lookup and did not change Ubuntu’s normal resolver settings.

I verified DNS with dig and the application response with curl as separate tests.

### Stopped the temporary DNS listener

```bash
sudo ss -lupn 'sport = :1053'
```

I identified dnsmasq as the listener and checked its current process ID.

For the final cleanup, the observed PID was 2225:

```bash
sudo kill -TERM 2225
```

This PID was specific to that running process and should not be reused without checking.

```bash
sudo ss -ltupn 'sport = :1053'
```

No listener rows remained.

```bash
curl --noproxy '*' --connect-timeout 5 http://127.0.0.1:8080/
```

Nginx still returned my personal lab webpage.

## 4. What I Verified

| Check | Actual result | What it established |
|---|---|---|
| Nginx status | active | The service was active at the time checked |
| Direct-IP request | Personal lab page returned | The local application responded on port 8080 |
| Public A-record query | NOERROR and seven answers | The resolver returned IPv4 address records |
| Nonexistent-name query | NXDOMAIN and zero answers | DNS reported that the queried name did not exist |
| Curl request to nonexistent name | Resolution timeout; exit 28 | Resolution did not finish within the configured limit |
| System IPv4 name lookup | No output; exit 2 | getent did not find the requested name |
| Explicit public DNS query | Answer from 1.1.1.1 | The explicitly selected server answered |
| Incorrect local record | NOERROR and 192.0.2.10 | A successful query can return an incorrectly configured address |
| Corrected local record | NOERROR and 127.0.0.1 | The lab DNS answer reflected the correction |
| Curl with explicit address override | Lab page returned; exit 0 | The application responded using the supplied address |
| Cleanup listener check | No rows for port 1053 | No UDP or TCP listener remained on that port |
| Application after cleanup | Lab page returned | Stopping the lab DNS process did not stop Nginx |

## 5. Problems and Fixes

### Curl timed out instead of returning the expected resolution error

The guide expected curl exit code 6. My request instead timed out during name resolution and returned exit code 28.

I compared this with dig, getent, and a direct-IP request.

I confirmed that the local application still responded, but I did not establish the cause of the difference between the lookup tools.

### Ctrl+C did not stop the lab DNS process

Ctrl+C displayed ^C without returning the command prompt.

I checked the specific lab process:

```bash
ps -p 2057 -o pid,args
```

After verifying that it was the intended dnsmasq process, I stopped it:

```bash
sudo kill -TERM 2057
```

The process stopped and the prompt returned.

The reason Ctrl+C did not stop it remains unverified. The process ID changed after restarting the program.

### No upstream servers warning

Dnsmasq reported that no upstream servers were configured.

This was expected because the isolated lab configuration intentionally supplied a local record without upstream servers. No correction was needed.

## 6. What I Learned

### My Dunkin explanation

DNS is like a directory containing Dunkin store names and addresses. It helps me find a location, but it does not check whether the store is serving coffee.

NOERROR means the DNS query completed without a reported DNS error. The directory can still contain an incorrect address. I might receive an address successfully and arrive at a warehouse instead of Dunkin.

NXDOMAIN means the directory answered that the requested name does not exist. The directory responded; it did not simply fail to answer.

A timeout means I did not receive a completed answer within my waiting limit. I cannot conclude from that alone whether the name exists.

Even with the correct address, the drive-through might be closed or unable to provide what I requested. Similarly, DNS can work while a network connection or application fails.

### Key terms

| Term | My explanation |
|---|---|
| A record | Maps a name to an IPv4 address |
| AAAA record | Maps a name to an IPv6 address |
| Resolver | Handles name lookups for applications |
| Upstream DNS server | Another DNS server the local resolver can consult |
| TTL | How long a DNS answer may be cached |
| NOERROR | The query completed without a reported DNS error; it does not guarantee a correct address or working website |
| NXDOMAIN | The response says the queried name does not exist |
| Timeout | The operation did not finish within the waiting limit |
| --resolve | Supplies a name-and-port address override for a curl command |

NOERROR can also appear with no records of the requested type. A name may exist but have no AAAA record, for example.

### Teach-it-back questions

**1. Does NOERROR prove a website works?**

No. It does not verify the application or confirm that the configured address is correct.

**2. How does NXDOMAIN differ from a timeout?**

NXDOMAIN is a DNS answer saying the name does not exist. A timeout means no completed answer arrived within the waiting limit.

**3. What did querying port 1053 accomplish?**

It explicitly queried my temporary lab DNS listener instead of the normal DNS service on port 53.

**4. What did the corrected dig result prove?**

The lab DNS server returned the corrected address, 127.0.0.1.

**5. Did curl --resolve prove Ubuntu automatically used my lab DNS server?**

No. It supplied the address directly and bypassed DNS lookup for that request.

## 7. Scope

This was a guided personal lab in a local Ubuntu VM.

I practiced DNS queries, response interpretation, a local record correction, separate application testing, and cleanup.

I did not administer production DNS, change public DNS records, configure cloud DNS, or configure Ubuntu to automatically use the temporary DNS server.

Safe to say now:
“Completed a guided personal DNS troubleshooting lab, corrected a local address mapping, and verified DNS answers separately from application responses.”

Do not claim yet:
Production DNS administration or independent operational expertise.
