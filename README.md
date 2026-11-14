# webserver-simple-security

A small Bash script that locks a Linux web server so it only accepts web traffic coming through **Cloudflare**.

When a site sits behind Cloudflare, attackers can still skip Cloudflare's protection by finding the server's real IP address and connecting to it directly. This script closes that gap: it uses `iptables` to drop every web request that does not come from Cloudflare's network. Visitors reach the site normally through your domain, and direct-IP access is blocked.

```
Visitor ──> Cloudflare (proxy, WAF, DDoS protection) ──> your server   allowed
Attacker ──────────────────────────────────────────────> your server   blocked
```

## Features

- Blocks direct-IP access to the web server on ports 80 and 443
- Allows traffic only from Cloudflare's published IPv4 ranges
- Saves the rules so they survive a reboot
- SSH (port 22) and other services outside the web ports are not affected
- No dependencies beyond `iptables`

## Requirements

- A Linux server (tested on Ubuntu and Debian) with root or `sudo` access
- Your domain added to Cloudflare with the **proxy enabled** (orange cloud) on the records that point to this server
- `iptables-persistent`, so the rules can be saved:

```bash
sudo apt install iptables-persistent
```

## Usage

**1. Back up your current firewall rules**, so you can roll back if needed:

```bash
sudo iptables-save > ~/iptables-backup.rules
```

**2. Download and run the script:**

```bash
wget https://raw.githubusercontent.com/elvir-k/webserver-simple-security/main/secure.sh
chmod +x secure.sh
sudo ./secure.sh
```

**3. Check that it works:**

- Opening `https://your-domain.com` should load as usual.
- Opening `http://YOUR-SERVER-IP` directly should time out.

## How it works

1. A `DROP` rule blocks all incoming TCP traffic to the web ports.
2. An `ACCEPT` rule is added for each Cloudflare IPv4 range, placed above the `DROP` rule, so Cloudflare's traffic gets through.
3. The rules are saved to `/etc/iptables/rules.v4`, which `iptables-persistent` loads on boot.

## Rolling back

Restore the backup from step 1:

```bash
sudo iptables-restore < ~/iptables-backup.rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4 > /dev/null
```

## Good to know

- **Run it once.** Running the script again adds the same rules a second time. To re-apply, roll back first.
- **Port range.** The rules use the range `80:443`, which covers every port from 80 to 443, not only those two. Any other service listening in that range will also only be reachable through Cloudflare. Ports outside it, such as 22 (SSH) or 8080, are not affected.
- **IPv4 only.** If your server has a public IPv6 address with a web server listening on it, add the same rules for IPv6 with `ip6tables`, using Cloudflare's [IPv6 ranges](https://www.cloudflare.com/ips-v6/).
- **Cloudflare's IP list can change.** The script uses the ranges published at [cloudflare.com/ips](https://www.cloudflare.com/ips/). Check that page now and then, and update the script if new ranges appear.
- **Real visitor IPs.** Because all requests now arrive from Cloudflare, your web server logs will show Cloudflare's IPs. To log real visitor IPs, enable `mod_remoteip` (Apache) or `real_ip_header CF-Connecting-IP` (Nginx).

## Extra protection

In the Cloudflare dashboard you can also:

- Set **SSL/TLS** to **Full (strict)**
- Turn on **Under Attack Mode** temporarily while your site is being attacked
- Add **WAF rules** for common attack patterns

## Related

Setting up a new server? [lamp-ubuntu](https://github.com/elvir-k/lamp-ubuntu) installs Apache, MySQL and PHP on Ubuntu in one command.
