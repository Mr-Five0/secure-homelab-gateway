# Troubleshooting Guide

## Overview

This document provides comprehensive troubleshooting guidance for common issues encountered when deploying and operating a self-hosted homelab stack using Docker, Traefik, Cloudflare Tunnel, and Fail2Ban. Each section addresses specific problems with detailed explanations, root causes, and step-by-step solutions.

The guide is organized by component and problem domain to facilitate quick reference and systematic troubleshooting.

---

## Table of Contents

1. [Architecture & Core Concepts](#architecture--core-concepts)
2. [Traefik Configuration](#traefik-configuration)
3. [Fail2Ban & Security](#fail2ban--security)
4. [Networking & Application Logic](#networking--application-logic)

---

## Architecture & Core Concepts

This section covers fundamental architectural misunderstandings and configuration errors that affect the entire stack.

### Issue 1: The "Double SSL" Trap (Tunnel to Traefik)

#### Problem Description

A common misconception is that Traefik must handle HTTPS termination even when deployed behind Cloudflare Tunnel. This leads to protocol mismatches, infinite redirects, and connection failures.

#### Root Cause

Cloudflare Tunnel (cloudflared) performs SSL/TLS termination at the edge. The connection from the Tunnel to Traefik within your private network should use HTTP, not HTTPS. When both layers attempt to handle SSL, it creates a conflict.

#### Symptoms

- Infinite redirect loops
- Connection timeouts
- SSL handshake failures
- 502 Bad Gateway errors

#### Solution

Configure Traefik and all service routers to use HTTP entrypoints and remove TLS-related labels.

**Correct Configuration Example:**

```yaml
# docker-compose.yml
labels:
  - "traefik.http.routers.gitlab.entrypoints=http"
  - "traefik.http.routers.gitlab.rule=Host(gitlab.example.com)"
  # No TLS label needed here
```

**Incorrect Configuration (Avoid):**

```yaml
# ❌ WRONG - Do not use this
labels:
  - "traefik.http.routers.gitlab.entrypoints=https"
  - "traefik.http.routers.gitlab.tls=true"
```

#### Additional Notes

- SSL/TLS is handled entirely by Cloudflare at the edge
- Internal communication between cloudflared and Traefik uses plain HTTP
- This is the expected and recommended configuration for Tunnel deployments

---

### Issue 2: Cloudflare Tunnel Host Header Configuration

#### Problem Description

Traefik routes requests based on the `Host` header, but Cloudflare Tunnel may not forward the expected Host header by default, resulting in 404 errors even when services are correctly configured.

#### Root Cause

Cloudflare Tunnel's default configuration may not preserve or forward the Host header that Traefik expects for routing decisions.

#### Symptoms

- 404 Not Found errors for correctly configured services
- Traefik logs show requests without proper Host headers
- Services are accessible via IP but not via domain names

#### Solution

Explicitly configure the "HTTP Host Header" in the Cloudflare Tunnel dashboard settings to match your domain.

**Steps:**

1. Navigate to Cloudflare Zero Trust Dashboard
2. Select your Tunnel
3. Go to the Public Hostname configuration
4. Set "HTTP Host Header" to match your domain (e.g., `gitlab.example.com`)
5. Save the configuration

#### Additional Notes

- This setting ensures the Host header is correctly forwarded to Traefik
- Each public hostname may require individual configuration
- Verify the Host header in Traefik access logs after configuration

---

### Issue 3: Redundant SSL Configuration

#### Problem Description

Configuring Let's Encrypt (ACME) certificates in Traefik while already using Cloudflare's Edge Certificates creates unnecessary complexity and potential conflicts.

#### Root Cause

When using Cloudflare Tunnel, SSL/TLS termination occurs at Cloudflare's edge. Configuring ACME in Traefik is redundant and can cause certificate management conflicts.

#### Symptoms

- Certificate renewal failures
- Unnecessary resource usage
- Configuration complexity
- Potential certificate conflicts

#### Solution

Remove all ACME and certificate resolver configurations from Traefik. Offload SSL/TLS handling entirely to Cloudflare.

**Remove from Traefik Configuration:**

```yaml
# ❌ Remove these sections from traefik.yml
certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: http
```

**Keep Only:**

- HTTP entrypoints
- Basic routing configuration
- No certificate-related settings

#### Additional Notes

- Cloudflare provides free SSL certificates at the edge
- No certificate management is required on your server
- This simplifies the overall architecture

---

## Traefik Configuration

This section addresses Traefik-specific configuration issues and routing problems.

### Issue 4: Traefik Dashboard Routing Configuration

#### Problem Description

Accessing the Traefik dashboard via a configured domain results in 404 errors due to incorrect routing rules and service references.

#### Root Cause

The Traefik dashboard requires:
1. A trailing slash in the URL path
2. Proper path prefix matching for both `/api` and `/dashboard/` endpoints
3. The router must reference the internal API service (`api@internal`)

#### Symptoms

- 404 errors when accessing `traefik.example.com`
- Dashboard partially loads but API calls fail
- Incorrect routing to backend services

#### Solution

Configure the dashboard router with correct path matching and service reference.

**Correct Configuration:**

```yaml
# docker-compose.yml
labels:
  - "traefik.http.routers.dashboard.rule=Host(traefik.example.com) && (PathPrefix(/api) || PathPrefix(/dashboard/))"
  - "traefik.http.routers.dashboard.service=api@internal"
  - "traefik.http.routers.dashboard.entrypoints=http"
```

**Key Points:**

- Use `PathPrefix(/dashboard/)` with trailing slash
- Include both `/api` and `/dashboard/` paths
- Reference `api@internal` service (not a Docker service)
- Access dashboard via `http://traefik.example.com/dashboard/` (note trailing slash)

#### Additional Notes

- The `api@internal` service is a special Traefik service that provides dashboard and API access
- Always use trailing slashes when accessing the dashboard
- Consider adding authentication middleware for production use

---

### Issue 5: Middleware Namespace Syntax

#### Problem Description

Referencing middleware defined in dynamic configuration files (e.g., `config.yml`) from Docker labels fails because the provider namespace is not specified.

#### Root Cause

Traefik supports multiple middleware providers (Docker labels, file-based, etc.). When referencing file-based middleware from Docker labels, you must specify the `@file` namespace to avoid ambiguity.

#### Symptoms

- Middleware not applied to routes
- Configuration errors in Traefik logs
- Services not receiving expected middleware behavior

#### Solution

Append `@file` to middleware names when referencing file-based middleware from Docker labels.

**Correct Syntax:**

```yaml
# ✅ CORRECT
labels:
  - "traefik.http.routers.service.middlewares=secure-headers@file"
```

**Incorrect Syntax:**

```yaml
# ❌ INCORRECT
labels:
  - "traefik.http.routers.service.middlewares=secure-headers"
```

#### Additional Notes

- `@file` indicates the middleware is defined in a file-based provider
- `@docker` is implicit for middleware defined in Docker labels
- Multiple middlewares can be chained: `middleware1@file,middleware2@file`

---

### Issue 6: Internal Port Mismatches

#### Problem Description

Changing a container's exposed port without updating the corresponding application configuration and Traefik load balancer settings causes routing failures.

#### Root Cause

Port configuration must be consistent across three layers:
1. Container's internal application port
2. Docker exposed port
3. Traefik load balancer port configuration

When these are misaligned, Traefik sends traffic to the wrong port.

#### Symptoms

- Connection refused errors
- Services appear healthy but are unreachable
- Traefik health checks fail

#### Solution

Ensure port consistency across all configuration layers.

**Example: GitLab on Port 8888**

1. **Application Configuration** (GitLab):
```ruby
# gitlab.rb
nginx['listen_port'] = 8888
```

2. **Docker Compose**:
```yaml
services:
  gitlab:
    ports:
      - "8888:8888"
```

3. **Traefik Labels**:
```yaml
labels:
  - "traefik.http.services.gitlab.loadbalancer.server.port=8888"
```

#### Verification Steps

1. Check container logs for the actual listening port
2. Verify Traefik can reach the service: `docker exec traefik wget -O- http://gitlab:8888`
3. Review Traefik access logs for connection errors

---

## Fail2Ban & Security

This section covers Fail2Ban configuration issues, log parsing problems, and security-related misconfigurations.

### Issue 7: JSON Log Parsing and Date Format Handling

#### Problem Description

Standard Fail2Ban regex patterns fail when parsing Traefik's JSON-formatted access logs due to variable field ordering and timestamp format differences.

#### Root Cause

JSON log formats have two characteristics that break traditional regex:
1. **Field Order Instability**: JSON fields can appear in any order (e.g., `Status` may appear before `IP`)
2. **Timestamp Format**: Traefik uses ISO 8601 format with timezone information

Traditional anchored regex patterns that assume fixed field positions will fail.

#### Symptoms

- Fail2Ban does not detect violations
- Filter test commands return no matches
- Security rules not triggered despite malicious activity

#### Solution

Use `datepattern` to explicitly match timestamps and employ unanchored, field-order-independent regex patterns.

**Correct Filter Configuration:**

```ini
# filter.d/traefik-auth.conf
[Definition]
datepattern = ^.*"time":"%%Y-%%m-%%dT%%H:%%M:%%S
failregex = ^.*"request_Cf-Connecting-Ip":"<HOST>".*"Status":(401|403)
ignoreregex =
```

**Key Points:**

- `datepattern` matches the timestamp field regardless of position
- Regex uses `^.*` to match any preceding fields
- Field matching is order-independent (e.g., `"request_Cf-Connecting-Ip":"<HOST>"`)
- Use `<HOST>` placeholder for IP extraction

#### Testing

Test your filter configuration:

```bash
fail2ban-regex /path/to/traefik/logs/access.log /etc/fail2ban/filter.d/traefik-auth.conf
```

#### Additional Notes

- Always use `Cf-Connecting-Ip` header for real client IPs behind Cloudflare
- Test filters with actual log entries before deploying
- Consider log rotation and file path configuration

---

### Issue 8: Configuration File Syntax - Inline Comments

#### Problem Description

Adding inline comments (using `#`) at the end of configuration lines in Fail2Ban `.conf` or `.local` files causes parsing errors because Fail2Ban interprets the comment as part of the configuration value.

#### Root Cause

Fail2Ban uses INI-style configuration parsing. Inline comments are not properly supported and are treated as part of the value, leading to incorrect file paths, regex patterns, or other configuration values.

#### Symptoms

- Fail2Ban fails to start
- Configuration errors in logs
- Filters or jails not loading
- Path not found errors

#### Solution

Never use inline comments in Fail2Ban configuration files. Place all comments on separate lines.

**Incorrect Syntax:**

```ini
# ❌ WRONG
logpath = /var/log/traefik/access.log  # Traefik access log
failregex = ^.*"Status":401  # Match 401 errors
```

**Correct Syntax:**

```ini
# ✅ CORRECT
# Traefik access log path
logpath = /var/log/traefik/access.log

# Match 401 unauthorized errors
failregex = ^.*"Status":401
```

#### Best Practices

- Use separate lines for all comments
- Group related configurations with section comments
- Document complex regex patterns with multi-line comments

---

### Issue 9: Overly Broad Regex Patterns (False Positives)

#### Problem Description

Using overly broad regex patterns that ban IPs for any 404 error on `.php` files results in false positives, blocking legitimate users of applications like Nextcloud or WordPress.

#### Root Cause

Many legitimate applications generate 404 errors during normal operation:
- Nextcloud may request non-existent PHP files during updates
- WordPress plugins may check for various PHP files
- Application migrations or updates trigger 404s

Banning all `.php` 404s catches legitimate traffic.

#### Symptoms

- Legitimate users getting banned
- High false positive rate
- Applications becoming inaccessible
- Support requests from blocked users

#### Solution

Use specific, targeted regex patterns that focus on known malicious paths and separate different types of violations into distinct filters.

**Incorrect Pattern (Too Broad):**

```ini
# ❌ TOO BROAD - Bans all PHP 404s
failregex = ^.*"request_Cf-Connecting-Ip":"<HOST>".*"Path":"/.*\.php".*"Status":404
```

**Correct Pattern (Specific):**

```ini
# ✅ SPECIFIC - Targets known bot paths
failregex = ^.*"request_Cf-Connecting-Ip":"<HOST>".*"Path":"/(wp-login\.php|xmlrpc\.php|\.env|wp-config\.php)".*"Status":404
```

**Recommended Approach:**

Separate filters for different violation types:

1. **Authentication Failures** (`traefik-auth.conf`):
   - Match 401/403 status codes
   - Focus on login endpoints

2. **Bot Scanning** (`traefik-botsearch.conf`):
   - Match 404s on known malicious paths
   - Exclude application-specific paths

3. **Rate Limiting** (`traefik-ratelimit.conf`):
   - Match 429 status codes
   - Track excessive request rates

#### Additional Notes

- Review Fail2Ban logs regularly to identify false positives
- Adjust ban times and find times based on your traffic patterns
- Whitelist trusted IPs if necessary
- Monitor ban effectiveness and adjust patterns accordingly

---

## Networking & Application Logic

This section addresses networking issues, IP propagation problems, and application-level configuration challenges.

### Issue 10: Real IP Address Propagation

#### Problem Description

Applications behind Traefik receive the Docker gateway IP or Traefik's container IP instead of the actual client IP address, breaking IP-based features like geolocation, logging, and security controls.

#### Root Cause

In a multi-layer proxy setup (Cloudflare → Tunnel → Traefik → Application), the original client IP must be explicitly propagated through each layer using trusted proxy configuration.

#### Symptoms

- All requests show the same IP (Docker gateway or Traefik IP)
- Application logs show incorrect client IPs
- IP-based features (geolocation, rate limiting) don't work
- Security tools can't identify actual clients

#### Solution

Configure trusted proxies at every layer of the stack.

**1. Traefik Configuration** (`traefik.yml`):

```yaml
entryPoints:
  http:
    forwardedHeaders:
      trustedIPs:
        - "127.0.0.1/32"
        - "10.0.0.0/8"
        - "172.16.0.0/12"
        - "192.168.0.0/16"
        # Cloudflare IP ranges (update regularly)
        - "173.245.48.0/20"
        - "103.21.244.0/22"
        # ... add all Cloudflare IP ranges
```

**2. Application Configuration Examples:**

**Nextcloud** (`config.php`):
```php
'trusted_proxies' => ['traefik'],  // Docker service name
```

**Trilium** (Environment variable):
```yaml
environment:
  - TRILIUM_TRUSTED_PROXY=true
```

**GitLab** (`gitlab.rb`):
```ruby
nginx['real_ip_trusted_addresses'] = ['172.16.0.0/12']  # Docker network
nginx['real_ip_header'] = 'X-Forwarded-For'
```

#### Verification

Check application logs to verify real IPs are being received:

```bash
# Check Traefik access logs
docker logs traefik | grep "Cf-Connecting-Ip"

# Check application logs
docker logs nextcloud | grep "Remote IP"
```

#### Additional Notes

- Cloudflare automatically adds `Cf-Connecting-Ip` header with real client IP
- Traefik forwards this as `X-Forwarded-For`
- Applications must be configured to trust and use these headers
- Keep Cloudflare IP ranges updated

---

### Issue 11: Rate Limiting Configuration for SPAs

#### Problem Description

Applying strict global rate limits breaks Single Page Applications (SPAs) like Trilium that make numerous simultaneous requests during initial page load, resulting in legitimate users being rate-limited.

#### Root Cause

SPAs typically:
- Load multiple resources simultaneously on page load
- Make frequent API calls for real-time updates
- Send multiple requests in quick succession for data fetching

Global rate limits designed for traditional web applications are too restrictive for SPAs.

#### Symptoms

- SPA pages fail to load completely
- API requests return 429 (Too Many Requests)
- Intermittent functionality issues
- Poor user experience

#### Solution

Customize rate limiting per service based on application characteristics. Use higher limits for SPAs or disable rate limiting for authenticated internal services.

**Traefik Rate Limiting Middleware** (`config.yml`):

```yaml
http:
  middlewares:
    # Standard rate limit for most services
    rate-limit-standard:
      rateLimit:
        average: 50
        burst: 100
        period: 1m

    # Higher rate limit for SPAs
    rate-limit-spa:
      rateLimit:
        average: 200
        burst: 500
        period: 1m

    # No rate limit for authenticated internal services
    rate-limit-auth:
      rateLimit:
        average: 1000
        burst: 2000
        period: 1m
```

**Service-Specific Application** (`docker-compose.yml`):

```yaml
services:
  trilium:
    labels:
      # Use SPA-optimized rate limit
      - "traefik.http.routers.trilium.middlewares=rate-limit-spa@file"
  
  nextcloud:
    labels:
      # Use standard rate limit
      - "traefik.http.routers.nextcloud.middlewares=rate-limit-standard@file"
  
  internal-api:
    labels:
      # Use high limit for authenticated service
      - "traefik.http.routers.internal-api.middlewares=rate-limit-auth@file"
```

#### Configuration Guidelines

- **SPAs**: `average: 200-500`, `burst: 500-1000`
- **Standard Web Apps**: `average: 50-100`, `burst: 100-200`
- **APIs**: `average: 100-200`, `burst: 200-500`
- **Authenticated Services**: Consider higher limits or IP-based whitelisting

#### Additional Notes

- Monitor rate limit effectiveness in Traefik metrics
- Adjust based on actual traffic patterns
- Consider combining with authentication middleware
- Use Fail2Ban for persistent violators rather than strict rate limits

---

## Additional Resources

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Fail2Ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [Create API Key Guide](Create_APIkey.md)

---

## Contributing

If you encounter issues not covered in this guide, please document them following the format used here and consider contributing to improve this troubleshooting resource.
