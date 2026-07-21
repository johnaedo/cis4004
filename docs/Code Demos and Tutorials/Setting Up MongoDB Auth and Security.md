---
share_cis4004: "true"
share_cop4331c: "true"
site-folder: docs/Code Demos and Tutorials/
---
> [!NOTE]
> Source:  Claude.ai, verified manually by executing the steps and validating the output.

Since you're migrating Budget Planner to MongoDB, this is worth locking down properly before exposing it. Here's the process:

## 1. Create an admin user first (before enabling auth)

Connect to MongoDB locally (auth isn't required yet):

```bash
mongosh
```

Switch to the `admin` database and create a user:

```javascript
use admin

db.createUser({
  user: "adminUser",
  pwd: passwordPrompt(),  // will prompt securely instead of typing it inline
  roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
})
```

`passwordPrompt()` avoids leaving the password in your shell history — worth using over typing it as a plain string.

You can also create an app-specific user scoped to just your Budget Planner database:

```javascript
use budgetplanner

db.createUser({
  user: "budgetApp",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "budgetplanner" } ]
})
```

## 2. Enable authorization in the config file

Edit `/etc/mongod.conf`:

```yaml
security:
  authorization: enabled
```

If you're planning to expose this on the firewall, also bind explicitly rather than to all interfaces unless you need to:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,YOUR_SERVER_IP
```

## 3. Restart MongoDB

```bash
sudo systemctl restart mongod
```

## 4. Test auth works

```bash
mongosh -u adminUser -p --authenticationDatabase admin
```

If it lets you in with the password and rejects connections without one, you're good.

## 5. Before opening the firewall — a few things worth doing at the same time

- **Enable TLS/SSL** for the connection, not just username/password. Without it, credentials go over the wire in the clear.
- **Restrict by IP** in your firewall rules (`ufw allow from <trusted-ip> to any port 27017`) rather than opening it to `0.0.0.0`. MongoDB has historically been a popular target for mass-scanning bots when left open with weak/no auth.
- **Use a strong, unique password** for each user, and store it in your `.env` / connection string rather than hardcoding it (this ties directly into the Mongoose connection setup you've been building).
- Consider whether you actually need direct external access at all — if only your app server talks to MongoDB, keeping it on a private network/VPC and never exposing port 27017 publicly is safer than authenticating + opening it.

Want help updating your Mongoose connection string/config to use the new credentials once you've got the user created?