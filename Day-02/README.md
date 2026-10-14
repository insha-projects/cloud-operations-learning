# Day 2 — Nginx Configuration Failure and Recovery

**Lab date:** October 14, 2026  
**Environment:** Local Ubuntu Server 26.04.1 LTS VM in VirtualBox  
**Access:** SSH from Windows Terminal  
**Scope:** Guided personal lab; no production systems or cloud resources  
**Study time:** 2 hours

## Objective

Practice diagnosing a Linux service failure using configuration
validation, service status and logs. Restore a working configuration
and verify that the service and webpage recover.

## 1. Verified the Healthy Baseline

Before making changes, I checked the service state:

```bash
systemctl is-active nginx
```

**Observed result:** `active`.

I requested the custom page:

```bash
curl http://127.0.0.1:8080/
```

**Observed response:**

```text
Insha personal Linux lab - Day 1
```

I validated the configuration:

```bash
sudo nginx -t
```

**Observed result:** Syntax was OK and the configuration test succeeded.

These checks established that the service was working before the fault.

## 2. Created a Known-Good Backup

I checked whether the backup already existed:

```bash
sudo ls -l /etc/nginx/cloud-lab.day2.backup
```

**Observed result:** No such file or directory.

I copied the working configuration:

```bash
sudo cp -a /etc/nginx/conf.d/cloud-lab.conf /etc/nginx/cloud-lab.day2.backup
```

The `-a` option preserves file attributes, including permissions.

I compared the original and backup:

```bash
sudo cmp /etc/nginx/conf.d/cloud-lab.conf /etc/nginx/cloud-lab.day2.backup
```

**Observed result:** No output, indicating matching file contents.

The backup provided a known-good configuration for recovery.

## 3. Introduced a Deliberate Configuration Error

I appended an instruction that Nginx does not recognize:

```bash
printf '\nBROKEN_LAB_DIRECTIVE;\n' | sudo tee -a /etc/nginx/conf.d/cloud-lab.conf > /dev/null
```

This changed only the lab configuration file. It did not change
the webpage content.

I tested the edited configuration:

```bash
sudo nginx -t
```

**Observed result:** Configuration validation failed because of
the unknown directive.

I requested the page again:

```bash
curl http://127.0.0.1:8080/
```

**Observed result:** The existing page still responded.

### Explanation

Editing a configuration file does not immediately change the settings
used by a running Nginx process. Nginx continued serving the page
with its previously loaded, valid configuration.

## 4. Reproduced a Service Outage

I deliberately restarted Nginx with the invalid configuration present:

```bash
sudo systemctl restart nginx
```

**Observed result:** The restart failed.

I checked the service state:

```bash
systemctl is-active nginx
```

**Observed result:** `failed`.

I tested the page with a limited connection and request timeout:

```bash
curl --connect-timeout 3 --max-time 5 http://127.0.0.1:8080/
```

**Observed result:** Curl reported error 7 and could not connect
to port 8080.

### Explanation

Restarting stopped Nginx. Its attempt to start again failed because
the startup configuration check rejected the invalid directive.
The broken configuration was not successfully applied.

## 5. Investigated the Failure

I inspected the service summary:

```bash
sudo systemctl status nginx --no-pager -l
```

- `--no-pager` displays output directly in the terminal.
- `-l` avoids shortening long lines.

I inspected the latest 30 journal entries for Nginx:

```bash
sudo journalctl -u nginx -n 30 --no-pager
```

- `-u nginx` selects the Nginx service.
- `-n 30` selects the latest 30 entries.

The output identified:

```text
unknown directive "BROKEN_LAB_DIRECTIVE" in /etc/nginx/conf.d/cloud-lab.conf:11
```

### Diagnosis

| Finding | Evidence |
|---|---|
| Service was unavailable | Nginx state was `failed`; curl could not connect |
| Startup validation failed | Service output showed the configuration test failed |
| Invalid instruction | `BROKEN_LAB_DIRECTIVE` |
| Affected file | `/etc/nginx/conf.d/cloud-lab.conf` |
| Location | Line 11 |

The logs connected the service failure to a specific configuration
error rather than merely reporting that the service was unavailable.

## 6. Restored Service

I restored the backup over the broken lab configuration:

```bash
sudo cp -a /etc/nginx/cloud-lab.day2.backup /etc/nginx/conf.d/cloud-lab.conf
```

I validated the restored configuration:

```bash
sudo nginx -t
```

**Observed result:** Syntax was OK and the configuration test succeeded.

A separate warning reported that systemd service definitions had
changed on disk. I refreshed systemd's definitions:

```bash
sudo systemctl daemon-reload
```

This command did not repair the Nginx website configuration.
Restoring the backup repaired that configuration. The reason for
the separate service-definition change was not investigated.

I started Nginx:

```bash
sudo systemctl start nginx
```

I verified its state:

```bash
systemctl is-active nginx
```

**Observed result:** `active`.

I verified the page response:

```bash
curl http://127.0.0.1:8080/
```

**Observed response:**

```text
Insha personal Linux lab - Day 1
```

Recovery was verified through both service state and application response.

## Verification Summary

| Stage | Configuration | Service/page result |
|---|---|---|
| Baseline | Valid | Active; custom page returned |
| After editing, before restart | Invalid file on disk | Existing page still returned |
| After restart attempt | Invalid | Service failed; connection failed |
| After restoring backup and starting | Valid | Active; custom page returned |

## What I Learned

- A configuration file on disk can differ from the configuration
  a running service is using.
- `nginx -t` validates configuration; it does not apply it.
- Restart stops the service and attempts to start it again.
- Invalid configuration can prevent Nginx from starting.
- Service status gives a summary; journal entries provide
  additional event and error details.
- A backup supports recovery to a known-good state.
- `daemon-reload` refreshes systemd service definitions;
  it is different from reloading Nginx configuration.
- An active service alone does not prove the expected page works.
  The application response must also be checked.

## Prevention

Validate configuration before applying changes. Keep a known-good
recovery copy and verify the actual page response after changes.

In this lab, validation had already detected the error. The restart
was performed deliberately to observe the resulting outage.

## Local Evidence

Screenshots are retained locally and are not required in this repository.

- `01-healthy-baseline.png` — if saved
- `02-invalid-configuration.png` — invalid configuration and page response
- `03-service-outage.png` — restart failure and connection failure
- `04-error-diagnosis.png` — error identifying the directive, file and line
- `05-service-restored.png` — successful validation and recovery checks

## Teach It Back

1. Why did the page work before restarting despite invalid configuration?
   My answer: Nginx was still using its previously loaded, valid configuration. Editing the file did not automatically apply the changes.

2. Which log message identified the cause?
   My answer: unknown directive "BROKEN_LAB_DIRECTIVE" in /etc/nginx/conf.d/cloud-lab.conf:11 identified the invalid instruction affected file and line number.
3. Why validate the restored backup before starting Nginx?
   My answer: To check that the restored configuration was valid before attempting to start the service.

4. Why check both service state and the page response?
   My answer: An active state shows the service is running, but does not prove it serves the correct page. Curl checks the actual response.

5. How does daemon-reload differ from reloading Nginx?
   My answer: systemctl daemon-reload refreshes systemd’s service definitions. systemctl reload nginx asks Nginx to reload its website configuration.

## Honest Labeling

**Safe to say now:** Diagnosed and recovered a deliberately induced
Nginx configuration failure in a guided personal Linux lab.

**Do not claim yet:** Production incident response, professional
on-call experience, business downtime reduction or cloud deployment.
