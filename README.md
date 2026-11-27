# Redbelly Node: Move to a New Domain Without Resyncing

This guide is for Redbelly **mainnet** node operators whose node domain has expired (or is about to expire) and who want to move the node to a new domain **on the same VPS**.

Your synced chain data is not touched. The node restarts once, for about a minute, and continues from the block where it stopped.

A helper script, [`switch-domain.sh`](switch-domain.sh), does the server work. Each step is one command, and every check prints `OK`, `TODO`, `WARN` or `FAIL`.

> [!NOTE]
> This is a community guide, not an official Redbelly guide. The steps were first done by hand on a real mainnet node in November 2026. The script was then tested on Ubuntu 24.04 against copies of the official installer setup (`rbn-installer.run`, systemd service `redbelly.service`), including service files with and without quotes around the certificate paths.

## The whole process

| Step | What you do | Command | Node downtime |
|---|---|---|---|
| [1](#step-1-get-a-new-domain-and-point-it-to-your-vps) | Get a new domain and point it to your VPS | (registrar website) | None |
| [2](#step-2-download-the-script-and-run-the-check) | Download the script and run the check | `check` | None |
| [3](#step-3-get-the-ssl-certificate) | Get the SSL certificate | `cert` | None |
| [4](#step-4-open-a-redbelly-support-ticket) | Open a Redbelly support ticket | (support portal) | None |
| [5](#step-5-switch-the-node-after-support-replies) | Switch the node, after support replies | `apply` | About 1 minute |
| [6](#step-6-check-that-the-node-is-syncing) | Check that the node is syncing | `status` | None |
| [7](#step-7-clean-up) | Remove the old certificate | `certbot delete` | None |

## Before you start

> [!WARNING]
> - **Never run `sudo ./rbn-installer.run -- --uninstall`.** According to the Redbelly docs, it deletes all synced blocks.
> - Don't delete or move the node's data folder (usually `/opt/redbelly/rbn`).
> - Never share your private keys, not in the ticket, not on Discord, not with anyone helping you. The script never prints them.

- **Timing:** Redbelly nodes check each other's TLS certificates. Once support changes your hostname, other nodes stop answering your node until it uses the certificate for the new domain (the log shows `received 0 responses`). So don't do Step 5 before support replies, and do it as soon as they do.
- **Governor nodes:** according to the Redbelly docs, a governor that is inactive for more than 5 minutes gets jailed and loses 20% of its stake, and planned downtime must be announced 3 days in advance. The switch takes about a minute, but mention the restart in your ticket anyway.
- **Scope:** this guide only covers a domain change on the same VPS. If you are also moving to a new VPS or IP, this guide is not enough.

---

## Step 1: Get a new domain and point it to your VPS

**First, check whether you can still renew the old domain.** Most registrars let you renew a domain for some time after it expires (the grace period; its length depends on the registrar). If you can renew it, do that instead: no ticket, no commands, no restart.

**If you need a new domain:**

- Buy it from any registrar, such as Namecheap, GoDaddy, Hostinger, Porkbun or Cloudflare. Check the renewal price before you buy; many domains are cheap only in the first year.
- Turn **auto-renew on**.
- A root domain (`mynode.com`) and a subdomain (`rbn.mynode.com`) both work.

In the registrar's DNS settings (or in Cloudflare, if your DNS is there), create this record:

| Type | Host / Name | Value | TTL |
|---|---|---|---|
| A | `@` (for a subdomain, its name, such as `rbn`) | Your VPS IP | Automatic or 300–600 |

- Delete any other A or AAAA records for the same name, such as the registrar's parking page.
- **Cloudflare:** set the record to "DNS only" (grey cloud). With the proxy on (orange cloud), node traffic never reaches your VPS.

DNS changes can take from a few minutes up to 24 hours. Step 2 tells you when the domain points to your VPS.

---

## Step 2: Download the script and run the check

SSH into your VPS and run:

```bash
cd ~
curl -fsSLO https://raw.githubusercontent.com/0xDarkSeidBull/redbelly-mainnet-node-domain-change-guide/main/switch-domain.sh
sudo bash switch-domain.sh check new-domain.com
```

Replace `new-domain.com` with your new domain, without `https://`. Run all later commands from the same folder (your home folder).

`check` changes nothing. It shows your current setup, checks that the new domain points to this server, and shows exactly which files the switch will change. To read the script before you run anything: `less switch-domain.sh`

Example output on a node installed with the official installer:

```text
== Node setup ==
  OK    Service file: /etc/systemd/system/redbelly.service
        Node ID 123, runs as user rbnuser
        Chain data: /opt/redbelly/rbn (this script never touches it)
  OK    Current certificate: old-domain.xyz, valid until Feb 22 13:38:27 2027 GMT
        /etc/redbelly/certs/old-domain.xyz/fullchain.pem

== New domain: new-domain.com ==
  OK    new-domain.com points to this server (203.0.113.10)
  TODO  No SSL certificate for new-domain.com yet
        Get it with: sudo bash switch-domain.sh cert new-domain.com

== Switch plan ==
        Old domain: old-domain.xyz
        New certificate goes to: /etc/redbelly/certs/new-domain.com/fullchain.pem
        New key goes to:         /etc/redbelly/certs/new-domain.com/privkey.pem
        Files where old-domain.xyz becomes new-domain.com:
          /etc/letsencrypt/renewal-hooks/deploy/redbelly-cert.sh
          /etc/letsencrypt/renewal-hooks/post/redbelly.sh
          /etc/redbelly/rbninstaller-config
          /etc/systemd/system/redbelly.service
  OK    Certificate renewals will keep the node's copy up to date
```

If you see `FAIL`, fix that item and run `check` again. The most common one is that the domain doesn't point to this server yet: wait a few minutes and try again.

---

## Step 3: Get the SSL certificate

```bash
sudo bash switch-domain.sh cert new-domain.com
```

The node keeps running during this step. The script:

- checks the DNS again first, because Let's Encrypt blocks new attempts for an hour after 5 failures;
- frees port 80 for a few seconds if Caddy, nginx, Apache or a Docker container is using it, and tells certbot to do the same at every renewal;
- runs certbot and checks the new certificate;
- prints the text for your support ticket.

You can add an email address at the end of the command (`... cert new-domain.com you@example.com`). It's optional.

If certbot says `command not found`, install it first: `sudo apt update && sudo apt install -y certbot`

---

## Step 4: Open a Redbelly support ticket

Step 3 printed a ticket text with your Node ID, old domain, new domain and VPS IP already filled in. It looks like this:

```text
Subject: Hostname change for Node ID 123

Hi team,

My node (Node ID: 123) is registered with hostname old-domain.xyz.
That domain has expired, so I have moved to new-domain.com, which points to the
same server IP 203.0.113.10. Ports, vote/signing address and reward address
are unchanged.

Please update the registered hostname for my node to new-domain.com and let me
know when it is done, so I can switch my node to the new certificate.
The switch needs a restart of about 1 minute.

Thanks
```

Send it as a ticket on Redbelly's **Node Operator Support Portal**. It's the same portal where you registered your node; the link is in Redbelly's onboarding email. You can also ask in the node operator channel on Redbelly's Discord. To print the text again, run `check` again.

---

## Step 5: Switch the node (after support replies)

> [!IMPORTANT]
> Run this only **after** support confirms that your hostname has been changed. Then run it right away.

```bash
sudo bash switch-domain.sh apply new-domain.com
```

The script runs all checks again and stops **without changing anything** if one of them fails. Then it asks you to type `yes`. After that it:

1. backs up the files it will change;
2. copies the new certificate to the node's certificate folder, with the same owner and permissions as the old one;
3. replaces the old domain with the new one in the service file, the installer config and the certbot renewal hooks;
4. checks, as the node's user, that the node can read the new certificate. If not, it puts every file back and stops without restarting;
5. restarts the node and waits until it logs its block number.

A successful run ends like this:

```text
== Switching ==
  OK    Backup of the current files: /root/rbn-backup-20261124-133828
  OK    Installed the new certificate in /etc/redbelly/certs/new-domain.com
  OK    Updated /etc/letsencrypt/renewal-hooks/deploy/redbelly-cert.sh
  OK    Updated /etc/letsencrypt/renewal-hooks/post/redbelly.sh
  OK    Updated /etc/redbelly/rbninstaller-config
  OK    Updated /etc/systemd/system/redbelly.service
  OK    Checked: the service file points to the certificate for new-domain.com and user rbnuser can read it
  Restarted. Waiting for the node to start (up to 3 minutes) ......
  OK    The node started and continues from block 3278304 (no resync)

== Sync ==
        Node block (from the log): 3278304, logged 0 min ago
        Network block:             3278306
  OK    The node is in sync with the network
```

Running `apply` a second time is safe: if everything is already switched, it says `Nothing to change` and does not restart the node.

---

## Step 6: Check that the node is syncing

After 10–15 minutes:

```bash
sudo bash switch-domain.sh status new-domain.com
```

```text
== Node status ==
  OK    redbelly.service is running (since Tue 2026-11-24 12:55:50 IST)
  OK    The service file uses the certificate for new-domain.com, valid until Feb 22 13:38:28 2027 GMT
  OK    The node serves the certificate for new-domain.com on port 1111

== Sync ==
        Node block (from the log): 3278304, logged 0 min ago
        Network block:             3278306
  OK    The node is in sync with the network
```

New blocks arrive slowly on mainnet (about one every 2 minutes in our test), so a difference of a few blocks is normal. The node's block number should keep going up.

---

## Step 7: Clean up

When `status` shows the node in sync, delete the old domain's certificate. `apply` printed this command with your old domain filled in:

```bash
sudo certbot delete --cert-name old-domain.xyz
```

Press `Y` when asked. If you skip this, certbot keeps trying to renew the dead domain once its certificate gets close to expiry.

After a few days without problems, you can also delete the backup folder that `apply` printed (`sudo rm -r /root/rbn-backup-...`). It contains a copy of your keys.

---

## If something goes wrong

**`check` or `cert` says the domain doesn't point to this server**

Fix the A record at your registrar (Step 1), wait a few minutes, and run the command again. If you use Cloudflare, make sure the record is "DNS only".

**`cert` fails**

The usual causes: the domain doesn't point to this server yet, port 80 is closed in a firewall (`sudo ufw allow 80`, and also check your VPS provider's firewall), or the Cloudflare proxy is on.

**`cert` says Caddy, nginx or Apache did not start again**

The certificate is fine, and the Redbelly node is not affected. Your web server failed to start because of an error in its own config. See why with `sudo journalctl -u caddy -n 30 --no-pager` (use your web server's name). For example, `ambiguous site definition: example.com` means that site is defined twice in `/etc/caddy/Caddyfile`. Remove one of the two blocks, then run `sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl start caddy`.

**`status` shows `received 0 responses` after the switch**

Wait a few minutes and run `status` again. If it doesn't stop, ask support to check that your hostname is exactly your new domain, and include the last lines of `/var/log/redbelly/rbn_logs/rbbc_logs.log`.

**The node doesn't start after `apply`**

The script prints the last lines of the node's error log. To put the old files back, run the `restore.sh` command it printed (see [Undo](#undo)).

**You used the first version of this guide and got `KEY READ: PROBLEM`, `Permission denied`, or the node doesn't start**

The first version (manual commands) broke on servers whose service file puts the certificate paths in quotes. It updated the service file but copied the certificate into a wrong folder named `"` in your home folder. You don't need to undo anything. Download the script (Step 2) and run:

```bash
sudo bash switch-domain.sh check new-domain.com
sudo bash switch-domain.sh apply new-domain.com
```

`apply` installs the certificate where the service file expects it and restarts the node. Then delete the wrong folder with the command the script prints (`sudo rm -r '/root/"'`). It contains a copy of your key.

**Support says "try syncing from the beginning"**

Run `check` and `status` first. Problems after a domain change come from the certificate or the config, and a resync from block 0 would use the same certificate and config. Only resync if support confirms that your data is corrupted.

---

## Undo

`apply` saves the old files in `/root/rbn-backup-<date>-<time>` and prints the undo command:

```bash
sudo bash /root/rbn-backup-YYYYMMDD-HHMMSS/restore.sh
```

This puts the old files back and restarts the node. Keep in mind that once support has changed your hostname, the node can't sync with the old certificate. Undo is for getting back to a known state, not a fix.

---

## What the script changes

- `check` and `status` only read. They change nothing.
- `cert` runs `certbot certonly --standalone` for the new domain. If a web server or Docker container uses port 80, it adds `--pre-hook`/`--post-hook` to stop and start it; certbot saves these for renewals.
- `apply`, after you type `yes`:
  - saves the service file, `/etc/redbelly` and `/etc/letsencrypt/renewal-hooks` in a backup folder;
  - copies `fullchain.pem` and `privkey.pem` from `/etc/letsencrypt/live/<new domain>/` to the node's certificate folder;
  - replaces the old domain with the new one only in files that contain it, and only where it is the whole domain name (`sub.old-domain.xyz` stays as it is);
  - restarts `redbelly.service`.
- It never touches the chain data, certbot's own settings, or the old certificate.

## Other setups

The script stops with an explanation if your node is not a systemd service called `redbelly.service`, or if it reads its certificate from a place the script can't update safely.

Older manual setups (for example the `observe.sh` setups from the devnet days) may use `/etc/letsencrypt/live/<domain>/` directly and may also have the domain in `config.yaml`. In that case:

1. Get the certificate with Step 3.
2. Find the files that contain the old domain:
   ```bash
   sudo grep -rnI --exclude-dir={rbn,database,chaindata} "old-domain.xyz" /etc /root /home 2>/dev/null
   ```
3. Replace the old domain with the new one in those files, make sure the node's user can read the new key, and restart the node.

If you're not sure, ask on Redbelly's Discord before you continue.

---

## Next time

- Keep auto-renew on at your registrar, and keep your payment method valid.
- Put the domain's expiry date in your calendar.
- Certbot renews the SSL certificate automatically, but you have to renew the domain yourself.

## Changes

- **24 Nov 2026:** the manual steps were replaced by `switch-domain.sh`. The first version of this guide broke on service files that put the certificate paths in quotes. If you ran it, see [If something goes wrong](#if-something-goes-wrong).

## References

- Redbelly docs: [Setup (DNS, firewall, uninstall)](https://vine.redbelly.network/nodes/operator-guide/setup/), [Start (certificates)](https://vine.redbelly.network/nodes/operator-guide/start/), [Verify (service and logs)](https://vine.redbelly.network/nodes/operator-guide/verify/), [Governance (jail rules)](https://vine.redbelly.network/nodes/governance/), [Governor requirements](https://vine.redbelly.network/nodes/reconfiguration/)
- Redbelly mainnet RPC: `https://governors.mainnet.redbelly.network` (chain ID 151)
- [Certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
