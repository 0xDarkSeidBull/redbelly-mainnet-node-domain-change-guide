# Redbelly Node: Switch to a New Domain After Your Domain Expires (Without Resyncing)

This guide is for Redbelly **mainnet** node operators whose node domain has expired (or is about to expire) and who want to move the node to a new domain **on the same VPS**.

Your synced chain data stays as it is. After the restart, the node continues from the block where it stopped. The node is down for only about one minute.

> [!NOTE]
> This is a community guide, not an official Redbelly guide. It was tested in November 2026 on a node installed with the official `rbn-installer.run` (systemd service `redbelly.service`).

## What changes and what doesn't

| Item | Changes? |
|---|---|
| Synced chain data (`/opt/redbelly/rbn`) | No |
| Node ID, keys, signing and reward addresses | No |
| VPS IP and ports | No |
| Hostname registered with Redbelly | Yes, through a support ticket |
| TLS certificate | Yes, a new one for the new domain |
| Certificate path in the service file, installer config and renewal hooks | Yes |

## Before you start

> [!WARNING]
> - **Never run `sudo ./rbn-installer.run -- --uninstall`.** According to the Redbelly docs, it deletes all synced blocks.
> - Do not delete or move the `--datadir` folder (usually `/opt/redbelly/rbn`).
> - Never share your private key with anyone: not in the ticket, not on Discord, not with anyone helping you.

- **Timing:** Redbelly nodes connect to each other with mutual TLS, which means every node checks the other node's certificate. In our test, support updated the hostname while the node was still using the old certificate. During that time the node got no response from any peer (the log showed `received 0 responses`). So don't switch before support replies, and don't wait after they reply. Do [Step 5](#step-5-switch-the-node-to-the-new-domain) as soon as the reply arrives.
- **Governor nodes:** according to the Redbelly docs, a governor that is inactive for more than 5 minutes gets jailed and loses 20% of its stake. Planned downtime must be announced 3 days in advance. The node is down for only about a minute in this guide, but mention the restart in your ticket anyway.
- **Scope:** this guide only covers changing the domain on the same VPS. If you are also moving to a new VPS or IP, this guide is not enough.

## The whole process

| Step | Task | Node downtime? |
|---|---|---|
| [0](#step-0-check-your-current-setup) | Check your current setup | No |
| [1](#step-1-get-a-new-domain) | Get a new domain | No |
| [2](#step-2-point-the-domain-to-your-vps-ip) | Point the domain to your VPS IP | No |
| [3](#step-3-get-an-ssl-certificate-for-the-new-domain) | Get an SSL certificate for the new domain | No |
| [4](#step-4-open-a-redbelly-support-ticket) | Open a Redbelly support ticket | No |
| [5](#step-5-switch-the-node-to-the-new-domain) | After support replies, switch the node to the new domain | About 1 minute |
| [6](#step-6-check-that-the-node-is-syncing) | Check that the node is syncing | No |
| [7](#step-7-clean-up) | Clean up | No |

---

## Step 0: Check your current setup

SSH into your VPS and run this. It only prints information and changes nothing:

```bash
UNIT=$(systemctl show -p FragmentPath --value redbelly.service)
echo "Service file: $UNIT"
grep -oE -- '--(nodeId|datadir|tls\.cert|tls\.key)=[^ ]+' "$UNIT"
sudo openssl x509 -noout -subject -enddate -in "$(grep -oP -- '--tls\.cert=\K\S+' "$UNIT")"
echo "VPS IP: $(curl -s -4 ifconfig.me)"
```

The output looks like this:

```text
Service file: /etc/systemd/system/redbelly.service
--datadir=/opt/redbelly/rbn
--nodeId=123
--tls.cert=/etc/redbelly/certs/old-domain.xyz/fullchain.pem
--tls.key=/etc/redbelly/certs/old-domain.xyz/privkey.pem
subject=CN = old-domain.xyz
notAfter=Jan 30 02:43:22 2027 GMT
VPS IP: 203.0.113.10
```

Note down:

- `--nodeId`: your Node ID. You need it for the ticket.
- `subject=CN = ...`: your old domain.
- `--datadir`: your synced data. Never touch this folder.
- `VPS IP`: you need it for DNS and for the ticket.

> [!IMPORTANT]
> This guide is for setups where the `--tls.cert` path starts with `/etc/redbelly/certs/`. The official installer creates this setup. If your path is different, read [Other setups](#other-setups) first.

---

## Step 1: Get a new domain

**First, check whether you can still renew the old domain.** Most registrars let you renew a domain for some time after it expires. This is called the grace period, and its length depends on the registrar. If you can renew, that is the easiest option: no ticket, no commands, no restart. After renewing, run `getent hosts old-domain.xyz` and check that the domain shows your VPS IP again.

**If you need a new domain:**

- Buy it from any registrar, such as Namecheap, GoDaddy, Hostinger, Porkbun or Cloudflare.
- Check the **renewal price** before you buy. Many domain extensions are cheap in the first year and expensive after that.
- Turn **auto-renew on** so this doesn't happen again.
- Both a root domain (`mynode.com`) and a subdomain (`rbn.mynode.com`) work.

---

## Step 2: Point the domain to your VPS IP

In your registrar's DNS panel (or in Cloudflare, if your DNS is there), create this record:

| Type | Host / Name | Value | TTL |
|---|---|---|---|
| A | `@` (for a subdomain, its name, such as `rbn`) | Your VPS IP | Automatic or 300–600 |

- Remove any old A or AAAA records for the same host, such as parking page records. If a wrong AAAA record is left, Let's Encrypt may check over IPv6 and the certificate request can fail.
- **Cloudflare:** keep the proxy OFF (grey cloud, "DNS only"). With the proxy on, the domain resolves to a Cloudflare IP and traffic on ports 1888/1111 never reaches your node.

Now check from the VPS. DNS changes can take from a few minutes up to 24 hours.

```bash
NEW=new-domain.com
getent ahosts "$NEW" | awk '{print $1}' | sort -u
curl -s -4 ifconfig.me; echo
```

The first command should show only your VPS IP, and it should match the IP from the second command. If you also added an AAAA record for your VPS's IPv6 address, that address will show up too. That's fine.

Ports 80 (for the certificate), 1888 and 1111 must be open in the firewall. This is normally done during installation. To check, run `sudo ufw status`.

---

## Step 3: Get an SSL certificate for the new domain

The node doesn't use port 80, so it keeps running during this step.

First, check what is running on port 80:

```bash
sudo ss -ltnp | grep ':80 '
```

**If nothing shows up (port 80 is free):**

```bash
NEW=new-domain.com
EMAIL=you@example.com
sudo certbot certonly --standalone -d "$NEW" --non-interactive --agree-tos -m "$EMAIL"
```

**If you see `caddy`, `nginx` or `apache2`:** certbot has to stop that service for a few seconds while it gets the certificate. Put your service name in `SVC`:

```bash
NEW=new-domain.com
EMAIL=you@example.com
SVC=caddy
sudo certbot certonly --standalone -d "$NEW" --non-interactive --agree-tos -m "$EMAIL" \
  --pre-hook "systemctl stop $SVC" --post-hook "systemctl start $SVC"
```

Certbot saves these hooks, so every auto-renewal does the same. If port 80 is used by a Docker container (`ss` shows `docker-proxy`), use `docker stop <container>` and `docker start <container>` in the hooks instead.

When the certificate is issued, you will see:

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/new-domain.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/new-domain.com/privkey.pem
```

- If you get `certbot: command not found`, install certbot first: `sudo apt update && sudo apt install -y certbot`
- If you see `Hook 'post-hook' reported error code 1` together with `Successfully received certificate`, the certificate is fine. Your other service (for example Caddy) just failed to start again. See [Troubleshooting](#troubleshooting).

---

## Step 4: Open a Redbelly support ticket

Open a ticket on Redbelly's **Node Operator Support Portal**. It's the same portal where you registered your node, and the link is in Redbelly's onboarding email. You can also get help in the node operator channel on Redbelly's Discord.

Send this message with your details filled in:

```text
Subject: Hostname change for Node ID <YOUR_NODE_ID>

Hi team,

My node (Node ID: <YOUR_NODE_ID>) is registered with hostname <OLD_DOMAIN>.
That domain has expired, so I have moved to <NEW_DOMAIN>, which points to the
same server IP <YOUR_VPS_IP>. Ports, vote/signing address and reward address
are unchanged.

Please update the registered hostname for my node to <NEW_DOMAIN> and let me
know when it is done, so I can switch my node to the new certificate. The
switch needs a restart of about 1 minute.

Thanks
```

As soon as support replies that the hostname has been updated, do Step 5.

---

## Step 5: Switch the node to the new domain

> [!IMPORTANT]
> Only do this step **after** support replies that the hostname has been updated. Run 5.1 to 5.3 in the same SSH window, because the variables set in 5.1 are used in the later steps.

### 5.1 Set the variables and check the paths

Put your domains in the first 2 lines only, without `https://` and without a trailing `/`:

```bash
OLD=old-domain.xyz
NEW=new-domain.com

UNIT=$(systemctl show -p FragmentPath --value redbelly.service)
SVC_USER=$(systemctl show -p User --value redbelly.service); SVC_USER=${SVC_USER:-root}
OLD_CERT=$(grep -oP -- '--tls\.cert=\K\S+' "$UNIT")
OLD_KEY=$(grep -oP -- '--tls\.key=\K\S+' "$UNIT")
NEW_CERT="${OLD_CERT//$OLD/$NEW}"
NEW_KEY="${OLD_KEY//$OLD/$NEW}"

printf '%-10s %s
' unit: "$UNIT" user: "$SVC_USER" old-cert: "$OLD_CERT" old-key: "$OLD_KEY" new-cert: "$NEW_CERT" new-key: "$NEW_KEY"
sudo ls -l "/etc/letsencrypt/live/$NEW/"
```

The output should look like this:

```text
unit:      /etc/systemd/system/redbelly.service
user:      rbnuser
old-cert:  /etc/redbelly/certs/old-domain.xyz/fullchain.pem
old-key:   /etc/redbelly/certs/old-domain.xyz/privkey.pem
new-cert:  /etc/redbelly/certs/new-domain.com/fullchain.pem
new-key:   /etc/redbelly/certs/new-domain.com/privkey.pem
```

Below that, the `ls` listing should show `fullchain.pem` and `privkey.pem`.

Continue only if:

- `old-cert` and `old-key` both start with `/etc/redbelly/certs/`
- `new-cert` and `new-key` contain your new domain
- the `ls` listing shows `fullchain.pem` and `privkey.pem`

If your paths don't contain the domain name at all (for example `/etc/redbelly/certs/fullchain.pem`), `new-cert` and `old-cert` will be the same. That works too, because 5.2 will replace the old files with the new certificate. In that case, check whether a script in `/etc/letsencrypt/renewal-hooks/` copies the certificate to `/etc/redbelly/certs/` after each renewal. If none does, you have to copy it yourself after every renewal, as the Redbelly docs describe.

### 5.2 Back up, install the new certificate and update the paths

The node does not restart in this step. Paste the whole block at once:

```bash
# 1) Backup: service file, /etc/redbelly (keys + config), renewal hooks. Chain data is not included.
BACKUP=/root/rbn-backup-$(date +%F-%H%M)
sudo mkdir -p "$BACKUP"
sudo cp -a "$UNIT" /etc/redbelly "$BACKUP"/
[ -d /etc/letsencrypt/renewal-hooks ] && sudo cp -a /etc/letsencrypt/renewal-hooks "$BACKUP"/
echo "Backup: $BACKUP"

# 2) Copy the new certificate into Redbelly's cert folder, with the same owner and permissions as the old files
sudo mkdir -p "$(dirname "$NEW_CERT")"
sudo cp -L "/etc/letsencrypt/live/$NEW/fullchain.pem" "$NEW_CERT"
sudo cp -L "/etc/letsencrypt/live/$NEW/privkey.pem" "$NEW_KEY"
sudo chown --reference="$(dirname "$OLD_CERT")" "$(dirname "$NEW_CERT")"
sudo chmod --reference="$(dirname "$OLD_CERT")" "$(dirname "$NEW_CERT")"
sudo chown --reference="$OLD_CERT" "$NEW_CERT"; sudo chmod --reference="$OLD_CERT" "$NEW_CERT"
sudo chown --reference="$OLD_KEY" "$NEW_KEY"; sudo chmod --reference="$OLD_KEY" "$NEW_KEY"

# 3) Replace the old domain with the new one in every file that contains it
FILES=$(sudo grep -rlI --exclude-dir=certs "$OLD" "$UNIT" /etc/redbelly /etc/letsencrypt/renewal-hooks 2>/dev/null)
echo "Files to update:"; echo "$FILES"
[ -n "$FILES" ] && sudo sed -i "s|$OLD|$NEW|g" $FILES

# 4) Check
echo "---- CHECK ----"
sudo openssl x509 -noout -subject -enddate -in "$NEW_CERT"
sudo -u "$SVC_USER" cat "$NEW_KEY" >/dev/null && echo "KEY READ: OK" || echo "KEY READ: PROBLEM"
grep -oE -- '--tls\.(cert|key)=[^ ]+' "$UNIT"
sudo grep -rnI --exclude-dir=certs "$OLD" "$UNIT" /etc/redbelly /etc/letsencrypt/renewal-hooks 2>/dev/null || echo "OLD DOMAIN LEFT: none"
```

On a setup made by the official installer, the updated files look something like this (your list may differ slightly):

```text
/etc/systemd/system/redbelly.service
/etc/redbelly/rbninstaller-config
/etc/letsencrypt/renewal-hooks/post/redbelly.sh
/etc/letsencrypt/renewal-hooks/deploy/redbelly-cert.sh
```

`rbninstaller-config` is the installer's settings file. It gets updated too, so that a future upgrade with the installer doesn't bring back the old path. The renewal hooks are updated so that after each renewal, the new certificate keeps getting copied to Redbelly's cert folder automatically.

Under `---- CHECK ----` you should see:

- `subject=CN = new-domain.com`
- `KEY READ: OK`
- the new domain in both `--tls.cert=` and `--tls.key=`
- `OLD DOMAIN LEFT: none`

If anything is different, don't restart the node. Check [Troubleshooting](#troubleshooting) first.

### 5.3 Restart the node

```bash
sudo systemctl daemon-reload
sudo systemctl restart redbelly.service
sleep 60
sudo systemctl status redbelly.service --no-pager | head -5
sudo tail -n 20 /var/log/redbelly/rbn_logs/rbbc_logs.log
sudo tail -n 10 /var/log/redbelly/rbn_logs/rbbc_logs_error.log
```

What to look for in the output:

- `Active: active (running)`.
- `current local block <N>` in the log, where N is a large number, not 0. The node continues syncing from this block. Example (your numbers will differ):
  ```text
  msg="Skipping download request ID 4 (range 3278304-3278304, is superblock false) with current local block 3278304 and superblock 2523450, ..."
  ```
- You may see `Node <ID> is changing from governor to candidate` and `Changed to candidate`. This means your node is not in the current governor list, so it runs as a candidate. Candidate nodes also keep the full chain in sync.
- If the error log only has warnings like `Flag --statusserver.addr has been deprecated`, everything is fine.

---

## Step 6: Check that the node is syncing

**Compare the node's block with the network's block:**

```bash
sudo grep -oE "(current local block|from) [0-9]{7,}" /var/log/redbelly/rbn_logs/rbbc_logs.log | tail -2
curl -s -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://governors.mainnet.redbelly.network \
  | python3 -c "import sys,json; print('network block:', int(json.load(sys.stdin)['result'],16))"
```

The two numbers should be close. New blocks arrive slowly on mainnet (about one every 2 minutes in our test), so a difference of a few blocks is normal. Run the commands again after 10–15 minutes. The node's number should keep moving up with the network.

**Check the certificate from outside:**

```bash
NEW=new-domain.com
echo | openssl s_client -connect "$NEW:1111" -servername "$NEW" 2>/dev/null | openssl x509 -noout -subject -enddate
```

The output should show `subject=CN = new-domain.com`. The node asks for a client certificate, so the connection closes right away. That's expected.

---

## Step 7: Clean up

Once the node is syncing properly, delete the old domain's certbot certificate. Replace `old-domain.xyz` with your old domain and press `Y` when asked:

```bash
sudo certbot delete --cert-name old-domain.xyz
```

If you skip this, certbot will try to renew the dead domain twice a day once the old certificate gets close to expiry, and it runs the renewal hooks on every attempt.

After a few days without problems, you can also remove:

- The old cert folder: `sudo rm -r /etc/redbelly/certs/old-domain.xyz`
- The backup from step 5.2 (`/root/rbn-backup-...`). It contains a copy of your private keys, so either keep it somewhere safe or delete it.

---

## Troubleshooting

**Certbot: `Could not bind TCP port 80 because it is already in use`**

Another service is using port 80. Use the command with hooks from Step 3.

**Certbot: `DNS problem: NXDOMAIN`, `unauthorized` or `Timeout during connect`**

Common causes: the domain doesn't point to your VPS IP yet, there's a wrong AAAA record, the Cloudflare proxy is on, or port 80 is closed in the firewall (`sudo ufw allow 80`). Go through Step 2 again.

**`Hook 'post-hook' reported error code 1` / `Job for caddy.service failed`**

You got the certificate. The problem is in your other service, which failed to start because of an error in its own config. To see why:

```bash
sudo journalctl -u caddy -n 30 --no-pager
```

Example: `ambiguous site definition: example.com` means that site is defined twice in the Caddyfile. Remove one of the two blocks, then run:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl start caddy
```

**Log shows `received 0 responses to get indices during block sync, but need at least N`**

The node gets no response from any peer. This is what you see after support has updated the hostname while the node still uses the old certificate. Do Step 5. If the message is still there after Step 5, run the CHECK from 5.2 again (in a new SSH session, run 5.1 first) and wait a few minutes. If it still doesn't recover, contact support and include the log.

**The node doesn't start, or the CHECK shows `KEY READ: PROBLEM`**

The user the node runs as (for example `rbnuser`) can't read the new key. Run the `chown`/`chmod --reference` lines from 5.2 again, then restart.

**Support says "try syncing from the beginning"**

Run the checks above first. Problems after a domain change usually come from the certificate or the config. A resync from 0 would use the same certificate and config, so the problem would stay. Only resync if support confirms that your data is corrupted.

### Other setups

Older manual setups (for example the `observe.sh` setups from the devnet days) may point straight to `/etc/letsencrypt/live/<domain>/`, and may also have the domain in `config.yaml`. The approach is the same:

1. Get the new certificate with Step 3.
2. Find the files that contain the old domain, such as the service file, a start script or `config.yaml`:
   ```bash
   sudo grep -rnI --exclude-dir={rbn,database,chaindata} "old-domain.xyz" /etc /root /home 2>/dev/null
   ```
   Replace the old domain with the new one in those files.
3. Make sure the user the node runs as can read the new key.
4. Restart the node and check it with Step 6.

If you're not sure, ask on Redbelly's Discord before you continue.

---

## Rollback

If a file got broken during 5.2, restore the backup. The backup folder name was printed after `Backup:` in the 5.2 output.

```bash
BACKUP=/root/rbn-backup-YYYY-MM-DD-HHMM
sudo cp -a "$BACKUP"/redbelly.service /etc/systemd/system/redbelly.service
sudo cp -a "$BACKUP"/redbelly/. /etc/redbelly/
sudo cp -a "$BACKUP"/renewal-hooks/. /etc/letsencrypt/renewal-hooks/
sudo systemctl daemon-reload && sudo systemctl restart redbelly.service
```

Keep in mind: support has already updated the hostname, so going back to the old certificate will not make the node sync. Rollback is only for fixing a broken file. The real fix is the new certificate.

---

## Next time

- Keep auto-renew on at your registrar, and keep your payment method valid.
- Put the domain's expiry date in your calendar.
- Certbot renews the SSL certificate automatically, but you have to renew the domain yourself.

## References

- Redbelly docs: [Setup (DNS, firewall, uninstall)](https://vine.redbelly.network/nodes/operator-guide/setup/), [Start (certificates)](https://vine.redbelly.network/nodes/operator-guide/start/), [Verify (service and logs)](https://vine.redbelly.network/nodes/operator-guide/verify/), [Governance (jail rules)](https://vine.redbelly.network/nodes/governance/), [Governor requirements](https://vine.redbelly.network/nodes/reconfiguration/)
- Redbelly mainnet RPC: `https://governors.mainnet.redbelly.network` (chain ID 151)
- [Certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
