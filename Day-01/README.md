# Day 1 — Linux Web Server Setup and Verification

**Lab dates:** October 11–12, 2026  
**Scope:** Guided personal lab; no production systems or cloud deployment

## 1. Objective

Set up a local Linux environment, configure Nginx to serve a custom
page, check file ownership and permissions, and verify service
operation after a restart.

## 2. Environment

| Component | Configuration |
|---|---|
| Host operating system | Windows 11 |
| Virtualization software | VirtualBox 7.2.20 r175154 |
| Guest operating system | Ubuntu Server 26.04.1 LTS |
| VM memory | 2048 MB |
| Virtual CPUs | 2 |
| Virtual disk | 25 GB |
| Networking | VirtualBox NAT |
| Administration | SSH from Windows Terminal |
| Web server | Nginx |
| Custom-page port | TCP 8080 |

The VM runs locally on my computer. It is not an AWS or Azure instance.

## 3. What I Did

### A. Installed Ubuntu in a Virtual Machine

I installed VirtualBox on Windows, created a VM and manually
installed Ubuntu Server.

I used Ubuntu Server as a lighter environment for my computer,
which has 8 GB RAM.

### B. Configured SSH Access

The VM console did not provide normal Windows copy/paste behavior.
I installed OpenSSH server inside Ubuntu:

```bash
sudo apt update
```

This refreshed Ubuntu's list of available packages and versions.
It did not upgrade installed applications.

```bash
sudo apt install -y openssh-server
```

This installed the SSH server. The `-y` option automatically
confirmed the package installation.

```bash
sudo systemctl enable --now ssh
```

This started the SSH service and enabled it to start at boot.

I configured a VirtualBox NAT forwarding rule:

| Setting | Value |
|---|---|
| Protocol | TCP |
| Host IP | 127.0.0.1 |
| Host port | 2222 |
| Guest IP | Left blank |
| Guest port | 22 |

Binding the forwarded port to Windows localhost limited access
to connections originating from my own computer.

From Windows Terminal, I connected using:

```powershell
ssh -p 2222 ubuntu_vm@127.0.0.1
```

VirtualBox forwarded this connection to the VM's SSH server.
After login, commands entered in this terminal ran inside Ubuntu.

### C. Checked the User and Operating System

```bash
whoami
```

This displayed the account running the command.

```bash
cat /etc/os-release
```

This displayed operating-system information.

**Observed results:**
- Logged-in user: `ubuntu_vm`
- Operating system: Ubuntu 26.04.1 LTS

### D. Installed and Started Nginx

```bash
sudo apt install -y nginx curl
```

This installed:
- **Nginx:** a web server that receives requests and serves content.
- **curl:** a command-line client for making requests and displaying responses.

```bash
sudo systemctl enable --now nginx
```

This started Nginx and enabled it to start when Ubuntu boots.

```bash
systemctl is-active nginx
```

**Observed result:** `active`.

I requested the default page:

```bash
curl http://127.0.0.1
```

**Observed result:** HTML containing “Welcome to nginx!”

Because no port was specified, this HTTP request used port 80.
The response came from Nginx's default sample page.

### E. Created a Separate Lab User and Web Directory

```bash
sudo adduser --disabled-password --gecos "" laboperator
```

This created the `laboperator` account without setting a usable
login password. `--gecos ""` left the optional profile fields blank.

```bash
sudo install -d -m 755 -o laboperator -g laboperator /var/www/cloud-lab
```

This created the web directory with:
- Owner: `laboperator`
- Group: `laboperator`
- Directory permissions: `755`

Under these permissions, the owner can read, modify and enter
the directory. Other users can list and enter it, but cannot
create or remove files in it through these permission bits.

### F. Created the Custom Page

```bash
sudo -u laboperator sh -c 'printf "Insha personal Linux lab - Day 1\n" > /var/www/cloud-lab/index.html'
```

This ran the quoted shell command as `laboperator` and wrote
the page text into `index.html`.

The `>` operator creates the file if absent or replaces its
contents if it already exists.

### G. Configured Nginx to Serve the Page

```bash
sudo tee /etc/nginx/conf.d/cloud-lab.conf > /dev/null <<'EOF'
server {
    listen 8080;
    server_name _;
    root /var/www/cloud-lab;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

This created a separate Nginx configuration for the lab.

| Configuration | Purpose |
|---|---|
| `listen 8080;` | Accept requests on TCP port 8080 |
| `server_name _;` | Set a placeholder server name; `_` is not a wildcard |
| `root /var/www/cloud-lab;` | Set the directory containing the page |
| `location /` | Define handling for matching request paths |
| `try_files $uri $uri/ =404;` | Check for a matching file or directory; otherwise return 404 |

For a request to `/`, Nginx's index handling served the
directory's `index.html` file.

The quoted `EOF` delimiter preserved the configuration text,
including `$uri`, without shell variable expansion.

### H. Validated and Applied the Configuration

```bash
sudo nginx -t
```

**Observed result:** Syntax was OK and the configuration test succeeded.

This checked the configuration before applying it.

```bash
sudo systemctl reload nginx
```

This asked running Nginx to load the configuration changes
without a full stop-and-start.

```bash
curl http://127.0.0.1:8080/
```

**Observed response:**

```text
Insha personal Linux lab - Day 1
```

This request ran inside Ubuntu and tested the custom page
through Ubuntu's loopback address on port 8080.

### I. Checked Ownership and Write Permissions

```bash
ls -ld /var/www/cloud-lab
```

This displayed details about the directory itself.

```bash
ls -l /var/www/cloud-lab/index.html
```

This displayed the file's permissions, owner, group, size
and modification time.

I checked write access for my login account:

```bash
test -w /var/www/cloud-lab/index.html
```

Immediately afterward:

```bash
echo $?
```

**Recorded result:** `1`, meaning the write test was false.

I checked write access as the file owner:

```bash
sudo -u laboperator test -w /var/www/cloud-lab/index.html
```

Immediately afterward:

```bash
echo $?
```

**Recorded result:** `0`, meaning the write test was true.

These tests checked access without modifying the file.

### J. Restarted Nginx and Verified Recovery

```bash
sudo systemctl restart nginx
```

This stopped Nginx and started it again.

```bash
systemctl is-active nginx
```

**Recorded result:** `active`.

```bash
curl http://127.0.0.1:8080/
```

**Recorded response:**

```text
Insha personal Linux lab - Day 1
```

The service and page checks confirmed operation after the restart.

## 4. What I Verified

| Check | Result |
|---|---|
| Ubuntu version | Ubuntu 26.04.1 LTS |
| SSH access | Connected from Windows Terminal and could paste commands |
| Initial Nginx state | Active |
| Default HTTP page | Returned the Nginx welcome page on port 80 |
| Custom configuration | Passed `nginx -t` |
| Custom page | Returned the lab text on port 8080 |
| Directory and page ownership | `laboperator` |
| Login-account write test | `1` — could not write |
| Lab-owner write test | `0` — could write |
| Service after restart | Active |
| Page after restart | Returned the custom page |

## 5. Problems and Fixes

### Copy/Paste Unavailable in the VM Console

**Problem:** Normal Windows copy/paste was unavailable in the
Ubuntu Server console.

**Action:** Installed OpenSSH server and configured a localhost-only
VirtualBox forwarding rule.

**Verification:** Connected through Windows Terminal and successfully
pasted commands into the Ubuntu SSH session.

## 6. What I Learned

### What Nginx Does

Nginx receives web requests and returns pages or files.
In this personal lab, it served the content of my `index.html`.

### What Configuration Controls

The configuration tells Nginx which port to listen on and
which directory contains the website files. My custom page
used port 8080 and `/var/www/cloud-lab`.

### Validation, Reload and Restart

- `nginx -t` checks configuration validity; it does not apply changes.
- Reload asks running Nginx to apply configuration changes.
- Restart stops the service and starts it again.

### Why “Active” Alone Is Insufficient

An active service does not prove it returns the correct page
on the intended port. I also made an HTTP request and checked
the returned content.

### Curl Output Versus an Exit Code

Curl displays the server's response. `echo $?` displays the
previous command's exit code.

For the write-permission tests, zero meant the test was true
and one meant it was false.

A successful curl command alone does not establish that the
expected page was returned; the response must also be inspected.

### Installed Versions Versus Available Versions

`dpkg-query -W nginx curl openssh-server` can report package
versions installed on the VM. Those are not necessarily the
software developers' latest releases.

## 7. Scope

This was a guided personal lab on a local virtual machine.

**Safe to say now:** Configured and verified a local Nginx
web service while practicing Linux users, permissions and
service management.

**Do not claim yet:** Production Linux administration,
professional cloud engineering, public website hosting
or enterprise server-support experience.
