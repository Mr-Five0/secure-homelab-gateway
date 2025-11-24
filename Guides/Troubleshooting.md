# Troubleshooting Guide

This guide summarizes critical architectural and configuration lessons learned during the deployment of a self-hosted stack.

## 🏗️ Architecture & Core Concepts

### 1. The "Double SSL" Trap (Tunnel to Traefik)
**The Misunderstanding:** Believing Traefik must handle HTTPS even when behind Cloudflare Tunnel.
**The Reality:** Cloudflare Tunnel decrypts traffic at the edge. The connection from the Tunnel (cloudflared) to Traefik inside your network should usually be HTTP, not HTTPS. Forcing HTTPS on Traefik's entrypoints causes protocol mismatches and infinite redirects.

*   **Mistake:** Enabling `entryPoints=https` and `tls=true` on Traefik routers behind a Tunnel.
*   **Fix:** Set Traefik and service entrypoints to `http` and remove TLS labels.

**Example (`docker-compose.yml`):**

```
✅ CORRECT
labels:
    - "traefik.http.routers.gitlab.entrypoints=http"
    - "traefik.http.routers.gitlab.rule=Host(gitlab.example.com)"

No TLS label needed here
```


### 2. Cloudflare Tunnel Host Headers
**The Issue:** By default, Cloudflare Tunnel might not forward the Host header expected by Traefik. Since Traefik routes based on `Host()`, requests fail (404).
**The Fix:** Explicitly configure the "HTTP Host Header" in the Cloudflare Tunnel dashboard settings to match your domain.

### 3. Redundant SSL Configuration
**The Issue:** Configuring Let's Encrypt (ACME) in Traefik while already using Cloudflare's Edge Certificates.
**The Fix:** Offload SSL to Cloudflare completely. Remove `certificatesResolvers` and `acme.json` from Traefik.

---

## ⚙️ Traefik Configuration

### 4. Traefik Dashboard Routing
**The Issue:** Accessing `traefik.example.com` fails with 404 because Traefik requires a trailing slash for the dashboard and must link to the internal API service.
**The Fix:** Always use `/` at the end of the URL (or redirect) and point the router to `api@internal`.

**Example:**
```
✅ CORRECT
labels:
    - "traefik.http.routers.dashboard.rule=Host(traefik.example.com) && (PathPrefix(/api) || PathPrefix(/dashboard/))"

    - "traefik.http.routers.dashboard.service=api@internal"
```

### 5. Middleware Namespace Syntax
**The Issue:** Referencing a middleware defined in a dynamic file (e.g., `config.yml`) inside a Docker label without specifying the provider namespace.
**The Fix:** Append `@file` to the middleware name.

**Example:**
*   ❌ Bad: `middlewares=secure-headers` (Looks for Docker middleware)
*   ✅ Good: `middlewares=secure-headers@file`

### 6. Internal Port Mismatches (GitLab)
**The Issue:** Changing a container's exposed port (e.g., to 8888) without updating the internal application port (Nginx) OR Traefik's load balancer port. Traefik continues sending traffic to port 80 by default.
**The Fix:** Ensure consistency across:
1.  Application config (e.g., `nginx['listen_port']`).
2.  Traefik label (`loadbalancer.server.port`).

---

## 🛡️ Fail2Ban & Security

### 7. JSON Log Parsing & Date Formats
**The Issue:** Standard regex fails on Traefik's JSON logs because field order is unstable (e.g., Status might appear before IP) and time formats differ.
**The Fix:**
1.  Use `datepattern` to explicitly match the timestamp field (e.g., `^.*"time":`).
2.  Use unanchored, minimal regex that doesn't rely on field order.

**Example (`filter.d/traefik.conf`):**
```
[Definition]
datepattern = ^.*"time":"%%Y-%%m-%%dT%%H:%%M:%%S
failregex = "request_Cf-Connecting-Ip":"<HOST>"
```

### 8. Configuration File Syntax ("Inline Comments")
**The Issue:** Adding comments (`#`) at the end of a configuration line in `.conf` or `.local` files. Fail2Ban interprets the comment as part of the filename/value.
**The Fix:** Never use inline comments in INI files. Place comments on their own lines.

### 9. overly Broad Regex (The "PHP" Ban)
**The Issue:** Banning any IP that generates a 404 on a `.php` file. Valid apps (Nextcloud, WordPress) often trigger 404s during normal operation, leading to false positives.
**The Fix:** Be specific. Ban known bot paths (`wp-login.php`, `.env`, `xmlrpc.php`) rather than *all* PHP files. Separate 404 (BotSearch) logic from 401/403 (AuthFailed) logic.

---

## 🌐 Networking & Application Logic

### 10. Real IP Propagation
**The Issue:** Applications see the Docker gateway IP or Traefik's IP instead of the user's IP.
**The Fix:** Configure "Trusted Proxies" at every layer.
1.  **Traefik:** Trust Cloudflare IPs (`forwardedHeaders.trustedIPs`).
2.  **Apps:** Configure apps (Nextcloud `trusted_proxies`, Trilium `trustedProxy=true`) to trust Traefik.

### 11. Rate Limiting Tuning
**The Issue:** Applying a global, strict rate limit breaks Single Page Applications (SPAs) like Trilium that fire many requests on load.
**The Fix:** Customize limits per service. Set higher `burst` and `average` values for heavy frontend apps, or disable rate limiting for authenticated internal services.
