# Hosting a Tor Hidden Service (.onion)

A Tor hidden service (officially called an **onion service**) lets you run a server that is accessible only via the Tor network at a `.onion` address, masking both your server's IP and your visitors' IPs.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Prerequisites](#prerequisites)
3. [Installing Tor](#installing-tor)
4. [Configuring a Web Server](#configuring-a-web-server)
5. [Configuring the Hidden Service](#configuring-the-hidden-service)
6. [Starting and Verifying](#starting-and-verifying)
7. [Vanity Addresses](#vanity-addresses)
8. [Security Hardening](#security-hardening)
9. [Operational Security (OpSec)](#operational-security-opsec)
10. [Troubleshooting](#troubleshooting)

---

## How It Works

Tor hidden services work through a **rendezvous circuit** system:

1. Your server picks **introduction points** in the Tor network and publishes a signed descriptor to a distributed hash table.
2. A client fetches the descriptor, picks a **rendezvous point**, and sends a one-time secret to your server via an introduction point.
3. Your server connects to the rendezvous point, completing a 6-hop circuit. Neither side learns the other's IP address.

The `.onion` address is a **56-character Base32 string** derived from the Ed25519 public key of your service (v3 onion services).

---

## Prerequisites

- A Linux server (Debian/Ubuntu used in examples below)
- Root or `sudo` access
- A web server (Nginx or Apache)
- Basic command-line familiarity

---

## simple version
install tor and nginx
```bash
sudo apt update
sudo apt install tor
sudo apt install nginx
```
edit tor config
```bash
sudo nano /etc/tor/torrc
```
uncomment thease lines
```config
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```
restart tor
```bash
sudo systemctl restart tor
```
get your .onion address
```bash
sudo cat /var/lib/tor/searxng/hostname
```
Remember to add your html to /var/www/html and read the config below to make sure its not on the clearnet

## Installing Tor

### Debian / Ubuntu

Add the official Tor Project repository for the most up-to-date version:

```bash
# Install dependencies
sudo apt install -y apt-transport-https curl gpg

# Add the Tor Project GPG key
curl -fsSL https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/tor-archive-keyring.gpg > /dev/null

# Add the repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] \
  https://deb.torproject.org/torproject.org $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/tor.list

# Install Tor
sudo apt update && sudo apt install -y tor deb.torproject.org-keyring
```

### Fedora / RHEL

```bash
sudo dnf install -y tor
```

---

## Configuring a Web Server

Your web server only needs to listen on **localhost** — it should never be exposed to the public internet.

### Nginx

```nginx
# /etc/nginx/sites-available/onion
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    root /var/www/onion;
    index index.html;

    # Hide server version
    server_tokens off;

    # Strip headers that could leak information
    proxy_hide_header X-Powered-By;
    add_header X-Frame-Options SAMEORIGIN;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/onion /etc/nginx/sites-enabled/
sudo mkdir -p /var/www/onion
echo "<h1>Hello from the dark side</h1>" | sudo tee /var/www/onion/index.html
sudo nginx -t && sudo systemctl reload nginx
```

### Apache

```apache
# /etc/apache2/sites-available/onion.conf
<VirtualHost 127.0.0.1:8080>
    DocumentRoot /var/www/onion
    ServerName localhost

    ServerTokens Prod
    ServerSignature Off

    <Directory /var/www/onion>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

```bash
sudo a2ensite onion
sudo systemctl reload apache2
```

---

## Configuring the Hidden Service

Edit `/etc/tor/torrc`:

```ini
# Basic v3 hidden service
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8080
```

- **`HiddenServiceDir`** — where Tor stores your private key and hostname.
- **`HiddenServicePort`** — maps the Tor-facing port (80) to your local service (127.0.0.1:8080).

You can expose multiple ports from the same service:

```ini
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80  127.0.0.1:8080
HiddenServicePort 443 127.0.0.1:8443
HiddenServicePort 22  127.0.0.1:22
```

Or run multiple distinct hidden services:

```ini
HiddenServiceDir /var/lib/tor/hs_web/
HiddenServicePort 80 127.0.0.1:8080

HiddenServiceDir /var/lib/tor/hs_ssh/
HiddenServicePort 22 127.0.0.1:22
```

---

## Starting and Verifying

```bash
# Reload Tor to apply the new configuration
sudo systemctl reload tor

# Your .onion address is now available:
sudo cat /var/lib/tor/hidden_service/hostname
# Example output:
# abcdef1234567890abcdef1234567890abcdef1234567890abcdefgh.onion
```

> **Note:** The first startup may take 1–3 minutes while Tor establishes introduction points. Subsequent starts are faster.

Test it from the **Tor Browser** by navigating to your `.onion` address.

---

## Vanity Addresses

V3 onion addresses are 56 characters. You can mine a prefix using **mkp224o**:

```bash
# Install build dependencies
sudo apt install -y gcc libsodium-dev make autoconf

# Clone and build
git clone https://github.com/cathugger/mkp224o.git
cd mkp224o
./autogen.sh && ./configure && make

# Generate addresses starting with "mysite"
./mkp224o mysite
```

> **Warning:** Each additional character multiplies search time by ~32. A 6-character prefix may take minutes; 8+ characters can take days or weeks on a single machine.

Once mkp224o finds a match, copy the generated folder (containing `hs_ed25519_secret_key`, `hs_ed25519_public_key`, and `hostname`) to your `HiddenServiceDir` with correct permissions:

```bash
sudo cp -r mysite<hash>/ /var/lib/tor/hidden_service/
sudo chown -R debian-tor:debian-tor /var/lib/tor/hidden_service/
sudo chmod 700 /var/lib/tor/hidden_service/
sudo systemctl reload tor
```

---

## Security Hardening

### Restrict Tor to only talk to localhost

Ensure Tor's `SocksPort` is not publicly exposed (it isn't by default):

```ini
# torrc
SocksPort 127.0.0.1:9050
```

### Firewall rules

Block all public inbound traffic except SSH (if needed for management):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp   # SSH for management — remove if not needed
sudo ufw enable
```

Your web server must **not** be reachable on any public port.

### Prevent web server from leaking your real IP

- Disable directory listings (`Options -Indexes` in Apache, no `autoindex on` in Nginx).
- Suppress `Server` and `X-Powered-By` headers.
- Avoid including external resources (Google Fonts, CDN scripts, analytics) — these cause the browser to make clearnet requests that bypass Tor.
- Disable PHP/app-level error messages that expose file paths or software versions.

### File permissions for the hidden service directory

```bash
sudo chown -R debian-tor:debian-tor /var/lib/tor/hidden_service/
sudo chmod 700 /var/lib/tor/hidden_service/
```

Tor will refuse to start the service if permissions are too loose.

---

## Operational Security (OpSec)

Running a hidden service is only as anonymous as your weakest link.

| Risk | Mitigation |
|---|---|
| IP leaked via server-side requests | Block outbound traffic from the web server process; audit all app code |
| Timing correlation attacks | Consider adding artificial jitter to responses for sensitive services |
| Metadata in served files | Strip EXIF data from images; remove author fields from documents |
| Server software fingerprinting | Keep software updated; suppress version banners |
| Physical access to the server | Use full-disk encryption; consider running from a dedicated machine |
| Key compromise | Back up `hs_ed25519_secret_key` securely offline; treat it like a root SSH key |
| Account linkage | Never access the server from a non-Tor connection; use separate identity-isolated accounts |

**Back up your private key.** Your `.onion` address is derived from it. If you lose `/var/lib/tor/hidden_service/hs_ed25519_secret_key`, your address is gone forever.

---

## Troubleshooting

### Tor fails to start after editing torrc

```bash
sudo tor --verify-config
```

This prints a human-readable error pointing to the offending line.

### Address not reachable after startup

- Wait 2–5 minutes for introduction points to propagate.
- Check Tor logs: `sudo journalctl -u tor --since "5 minutes ago"`
- Verify the local web server is listening: `ss -tlnp | grep 8080`
- Confirm `HiddenServiceDir` has correct ownership and `700` permissions.

### "Permission denied" errors in Tor logs

```bash
sudo chown -R debian-tor:debian-tor /var/lib/tor/
sudo chmod 700 /var/lib/tor/hidden_service/
```

### Web server returns 502/connection refused

The Tor service is up but the backend isn't listening on the expected local port:

```bash
# Check what's actually listening
ss -tlnp
curl -v http://127.0.0.1:8080/
```

---

## Further Reading

- [Tor Project — Onion Services Overview](https://community.torproject.org/onion-services/)
- [Tor Project — torrc Configuration Reference](https://2019.www.torproject.org/docs/tor-manual.html.en)
- [mkp224o — Vanity Address Generator](https://github.com/cathugger/mkp224o)
- [SecureDrop](https://securedrop.org/) — a real-world, hardened hidden service deployment for whistleblowers
