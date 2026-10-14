# Day 1 — Linux Web Server Personal Lab

## What I built
A local Ubuntu Server VM running Nginx, configured to serve my custom index.html page on port 8080.

## What I verified
- Ubuntu version: The VM reported Ubuntu 26.04.1 LTS.
- Configuration test: `sudo nginx -t` reported that the syntax was OK and the configuration test was successful.
- Custom-page response: A request to http://127.0.0.1:8080/ returned “Insha personal Linux lab - Day 1”.
- File ownership: The directory and index.html were owned by laboperator.
- My account's write test: Returned 1, meaning my account cannot write to index.html.
- laboperator's write test: Returned 0, meaning laboperator can write to index.html.
- Service after restart: Nginx reported active.
- Page after restart: The request returned Insha personal Linux lab - Day 1.

## What I learned
Write in my own words:
- What Nginx does: Nginx is a web server that receives requests and returns pages or files. In my personal lab, it served the page I created in index.html.
- What configuration controls: The configuration tells Nginx which port to listen on and which folder contains the website files. I configured port 8080 and /var/www/cloud-lab.
- Reload versus restart: Reload asks Nginx to apply configuration changes without fully stopping the service. Restart stops the service and starts it again. I used sudo nginx -t to validate the configuration before reloading.
- Why "active" alone is insufficient: An active service does not prove that it returns the correct page on the intended port. I also requested my page with curl and checked its response.
- curl output versus an exit code: Curl displays the response from the web server. echo $? displays the previous command’s exit code: zero means success and a nonzero value indicates an error. I checked the response text to confirm that my custom page was returned.

## Problem solved
Copy/paste was unavailable in the VM console.
I connected through SSH from Windows Terminal.

## Scope
Guided personal lab; no production or cloud deployment.
