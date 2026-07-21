---
share_cis4004: "true"
share_cop4331c: "true"
site-folder: docs/Code Demos and Tutorials/
---
>[!NOTE]
>Source:  Claude.ai, manually verified on a DigitalOcean droplet created with the MERN template.
# Securing MongoDB: Authentication + TLS Setup Guide

This document covers the full process for locking down a MongoDB instance before exposing it on the firewall:

1. Creating admin and application users
2. Enabling TLS/SSL with Let's Encrypt certificates
3. Handling the "chain of trust" requirement
4. Final `mongod.conf` configuration
5. Certificate renewal automation

> **Note:** MongoDB's config file is normally located at `/etc/mongod.conf` (not `/etc/mongodb.conf`) on most package installs. Adjust paths below if yours differs.

---

## 1. Create Users (before enabling auth)

Auth isn't required yet at this point, so connect locally without credentials:

```bash
mongosh
```

### Create an admin user

```javascript
use admin

db.createUser({
  user: "adminUser",
  pwd: passwordPrompt(),  // prompts securely, avoids leaving password in shell history
  roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
})
```

### Create an app-specific user (recommended)

Scope this to just your application's database rather than granting broad access:

```javascript
use budgetplanner

db.createUser({
  user: "budgetApp",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "budgetplanner" } ]
})
```

---

## 2. Obtain a TLS Certificate with Certbot

MongoDB validates TLS against a hostname, so this requires a real domain pointing at the server (not just an IP).

```bash
sudo certbot certonly --standalone -d yourdomain.example.com
```

Use `--webserver` mode instead of `--standalone` if nginx/apache is already running on port 80.

This creates the following files under `/etc/letsencrypt/live/yourdomain.example.com/`:

- `fullchain.pem` — full certificate chain
- `privkey.pem` — private key
- `chain.pem` — intermediate CA certificate (needed for the chain-of-trust requirement below)

---

## 3. Build the Certificate Files MongoDB Needs

MongoDB requires two separate files:

- A combined cert + key file (`certificateKeyFile`)
- A CA file for chain-of-trust validation (`CAFile`) — **required** as of MongoDB 4.4.29 / 5.0.25 / 6.0.14 / 7.0.6+ due to a security fix (CVE-2024-1351). Without this, `mongod` will refuse to start with the error _"The use of TLS without specifying a chain of trust is no longer supported."_

```bash
sudo mkdir -p /etc/ssl/mongodb

# Combined cert + private key
sudo bash -c 'cat /etc/letsencrypt/live/yourdomain.example.com/fullchain.pem \
  /etc/letsencrypt/live/yourdomain.example.com/privkey.pem \
  > /etc/ssl/mongodb/mongodb.pem'

# CA chain file, copied into a location the mongodb user can read
sudo cp /etc/letsencrypt/live/yourdomain.example.com/chain.pem /etc/ssl/mongodb/chain.pem

# Lock down permissions
sudo chown mongodb:mongodb /etc/ssl/mongodb/mongodb.pem /etc/ssl/mongodb/chain.pem
sudo chmod 600 /etc/ssl/mongodb/mongodb.pem /etc/ssl/mongodb/chain.pem
```

Certbot's `/etc/letsencrypt/live/` directory is normally root-only, which is why the files are copied into `/etc/ssl/mongodb` where the `mongodb` user can read them.

---

## 4. Update `/etc/mongod.conf`

Combine authentication, network binding, and TLS settings into the config file:

```yaml
security:
  authorization: enabled

net:
  port: 27017
  bindIp: 127.0.0.1,YOUR_SERVER_IP
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/mongodb.pem
    CAFile: /etc/ssl/mongodb/chain.pem
    allowConnectionsWithoutCertificates: true
```

**Field notes:**

- `authorization: enabled` — requires username/password for all connections
- `bindIp` — bind to localhost plus the server's actual IP rather than all interfaces (`0.0.0.0`), to limit exposure
- `tls.mode: requireTLS` — rejects any non-TLS connection
- `certificateKeyFile` — the combined server cert + key
- `CAFile` — required chain-of-trust file (intermediate CA)
- `allowConnectionsWithoutCertificates: true` — this is server-side TLS only (encrypts the connection); it does **not** require clients to present their own certificate. Client identity is handled via username/password instead. Omit this only if you want full mutual TLS (client certs), which is a more involved, separate setup.

> **YAML indentation matters.** `tls:` must be nested one level under `net:`, using spaces, not tabs — a common source of "unrecognized option" errors.

---

## 5. Restart and Verify

```bash
sudo systemctl restart mongod
sudo journalctl -u mongod -n 50   # check for startup errors
```

Test the connection with both auth and TLS:

```bash
mongosh --tls --host yourdomain.example.com -u adminUser -p --authenticationDatabase admin
```

If it connects successfully with the password and refuses connections without one, the setup is working.

---

## 6. Automate Certificate Renewal

Let's Encrypt certificates expire every 90 days. Certbot auto-renews the underlying cert, but MongoDB needs its combined PEM and CA file rebuilt, and the service restarted, after each renewal.

Create a deploy hook:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/mongodb-tls.sh
```

Contents:

```bash
#!/bin/bash
DOMAIN="yourdomain.example.com"

cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem \
    /etc/letsencrypt/live/$DOMAIN/privkey.pem \
    > /etc/ssl/mongodb/mongodb.pem

cp /etc/letsencrypt/live/$DOMAIN/chain.pem /etc/ssl/mongodb/chain.pem

chown mongodb:mongodb /etc/ssl/mongodb/mongodb.pem /etc/ssl/mongodb/chain.pem
chmod 600 /etc/ssl/mongodb/mongodb.pem /etc/ssl/mongodb/chain.pem

systemctl restart mongod
```

Make it executable:

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/mongodb-tls.sh
```

Certbot automatically runs every script in `renewal-hooks/deploy/` after a successful renewal, so this keeps MongoDB's certificates current without manual intervention.

---

## 7. Before Opening the Firewall

- Restrict the port to trusted IPs only, rather than opening it broadly:
    
    ```bash
    sudo ufw allow from <trusted-ip> to any port 27017
    ```
    
- Use strong, unique passwords per user, stored in environment variables / connection strings rather than hardcoded.
- Consider whether external exposure is even necessary — if only your app server talks to MongoDB, keeping it on a private network/VPC and never opening 27017 publicly is the safest option.

### Example Node.js / Mongoose connection string

```
mongodb://budgetApp:PASSWORD@yourdomain.example.com:27017/budgetplanner?tls=true&authSource=budgetplanner
```