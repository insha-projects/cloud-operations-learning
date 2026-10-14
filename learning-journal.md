## What I learned

Editing a configuration file does not immediately change the
configuration Nginx is using. The page initially continued working
because Nginx was still running with its previously loaded,
valid configuration.

When I restarted Nginx, the service stopped and attempted to start
again. Its startup configuration check rejected the invalid directive,
preventing the service from starting. Requests to the page then failed.

I used systemctl status and journalctl to investigate. The logs
identified BROKEN_LAB_DIRECTIVE in
/etc/nginx/conf.d/cloud-lab.conf at line 11.

I restored the known-good backup, validated it with sudo nginx -t,
and started Nginx. I verified recovery by checking that the service
was active and that curl returned my custom page.

The main lesson was to validate configuration before applying changes,
use logs to identify failures, and verify both service state and the
actual page response after recovery.
