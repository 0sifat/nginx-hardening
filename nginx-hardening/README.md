# Modular Nginx Hardening

Validated with `nginx -t` (nginx 1.24) and `gixy` (0 issues).

```
nginx.conf                         -> /etc/nginx/nginx.conf            (http globals)
conf.d/rate-limit-zones.conf       -> /etc/nginx/conf.d/               (http: limit zones)
conf.d/real-ip.conf.example        -> rename to .conf if behind ALB/Cloudflare
snippets/request-limits.conf       (http)     body size, buffers, timeouts
snippets/ssl-params.conf           (server)   TLS 1.2/1.3, ciphers, sessions
snippets/security-headers.conf     (server)   HSTS, CSP, nosniff, frame, etc.
snippets/rate-limit.conf           (server)   apply general req/conn limits
snippets/limit-methods.conf        (server)   GET/HEAD/POST only
snippets/deny-sensitive.conf       (server)   .git, .env, backups, vendor
snippets/proxy-params-secure.conf  (location) safe reverse-proxy defaults
sites-available/app.conf           example vhost
sites-available/default-deny.conf  catch-all for unknown Host / IP scans
```

## Install

```bash
sudo cp -r /etc/nginx /etc/nginx.bak.$(date +%F)
sudo cp nginx.conf /etc/nginx/
sudo cp snippets/*.conf /etc/nginx/snippets/
sudo cp conf.d/*.conf /etc/nginx/conf.d/
sudo cp sites-available/*.conf /etc/nginx/sites-available/
sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048

sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/default-deny.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/

sudo vim /etc/nginx/sites-available/app.conf   # set domain, cert paths, upstream port
sudo nginx -t && sudo systemctl reload nginx
```

## Check before going live

- `listen ... ssl http2` is used for nginx 1.24 compatibility. On 1.25.1+ switch to `http2 on;` to silence the deprecation warning.
- `ssl_reject_handshake` needs nginx 1.19.4+. Ubuntu 22.04 ships 1.18.
- HSTS: confirm every subdomain serves HTTPS first. Add `preload` only on purpose.
- CSP ships as Report-Only. Watch the browser console, then enforce it.
- `limit-methods.conf`: REST APIs need PUT/PATCH/DELETE/OPTIONS added.
- Behind an ALB/CDN: enable `real-ip.conf`, or every client shares one rate-limit bucket.
- OCSP stapling is off because Let's Encrypt retired OCSP in Aug 2025. Turn it on only for commercial CAs.

## Audit

```bash
sudo nginx -t
pip install gixy-ng && gixy /etc/nginx/nginx.conf
# External: ssllabs.com/ssltest, securityheaders.com
```
