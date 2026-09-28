# Minimal Private VPN + Telegram Proxy — Install Guide

One German VPS, ~€5–8/month. For family members in Russia. State of knowledge: September 2026.

**Legend:** *(unverified)* = reported, not confirmed; *(estimate)* = my approximation. UI labels in 3x-ui / client apps may differ slightly between versions.

---

## 0. What you end up with

| # | Service | Port | Role | Required? |
|---|---|---|---|---|
| 1 | VLESS + REALITY + Vision (self-steal) | 443/tcp | Main connection | Yes |
| 2 | VLESS + XHTTP + REALITY | 8443/tcp | Backup if #1 freezes | Yes |
| 3 | 3x-ui subscription server | 2096/tcp | One link per person, auto-updates configs | Yes |
| 4 | Hysteria2 (Salamander obfuscation) | 47000/udp | Backup if TCP is throttled | Optional |
| 5 | MTProto proxy (mtg ≥ 2.2.8, fake-TLS) | 2443/tcp | Telegram without VPN, mostly home Wi-Fi | Optional |

Internal only: nginx with a real website on `127.0.0.1:9443` (the "camouflage" site REALITY forwards probes to), 3x-ui panel (via SSH tunnel only).

Placeholders used below — replace everywhere:

| Placeholder | Meaning |
|---|---|
| `SERVER_IP` | VPS IPv4 |
| `vpn.example.com` | Your (sub)domain |
| `admin` | Your non-root Linux user |

---

## 1. Buy server and domain (≈30 min)

1. **VPS:** netcup VPS 500 (~€7/mo) or Contabo Cloud VPS 10 (~€4–5/mo) *(prices reported, verify)*. Location: Germany. OS: **Ubuntu 24.04**. Pay 1 month first — test before committing.
2. **Domain:** any cheap non-.ru domain (.xyz, .site, .online) at an international registrar (Porkbun, Namecheap, Cloudflare Registrar, etc.).
3. **DNS:** create an `A` record `vpn.example.com → SERVER_IP`, **TTL 300** (so you can move the server fast). If using Cloudflare DNS: **DNS only (grey cloud)**, not proxied.
4. Check it resolves before continuing:
   ```bash
   dig +short vpn.example.com   # must print SERVER_IP
   ```

---

## 2. Base server setup and hardening (≈20 min)

From your computer (generate a key first if you have none: `ssh-keygen -t ed25519`):

```bash
ssh root@SERVER_IP
```

On the server:

```bash
apt update && apt -y full-upgrade
apt -y install curl ufw fail2ban unattended-upgrades nginx certbot dnsutils

# Non-root user with sudo
adduser admin
usermod -aG sudo admin
```

From your computer, copy your SSH key, then test login in a **new** terminal:

```bash
ssh-copy-id admin@SERVER_IP
ssh admin@SERVER_IP
```

Only after key login works, disable password and root login:

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Firewall (SSH first, or you lock yourself out):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp      # certificate renewal + camouflage site
sudo ufw allow 443/tcp     # VLESS REALITY
sudo ufw allow 8443/tcp    # VLESS XHTTP
sudo ufw allow 2096/tcp    # subscription links
sudo ufw allow 47000/udp   # Hysteria2 (optional)
sudo ufw allow 2443/tcp    # MTProto (optional)
sudo ufw enable
```

Automatic security updates and TCP BBR (better speed on long routes):

```bash
sudo dpkg-reconfigure -plow unattended-upgrades   # answer "Yes"
echo -e "net.core.default_qdisc=fq\nnet.ipv4.tcp_congestion_control=bbr" | sudo tee /etc/sysctl.d/99-bbr.conf
sudo sysctl --system
```

---

## 3. Camouflage website + real certificate (≈15 min)

REALITY forwards anyone who is not your client (e.g. a DPI probe) to a real website. Using **your own domain and your own site** ("self-steal") avoids the SNI/IP mismatch that Russian DPI reportedly checks.

Put any simple page in place (a personal homepage, a "coming soon" page):

```bash
echo '<!doctype html><html><head><title>Coming soon</title></head><body><h1>Coming soon</h1></body></html>' | sudo tee /var/www/html/index.html
```

Get a Let's Encrypt certificate (nginx default site already serves `/var/www/html` on port 80):

```bash
sudo certbot certonly --webroot -w /var/www/html -d vpn.example.com \
  --deploy-hook "systemctl reload nginx" --agree-tos -m you@example.com -n
```

Create the internal HTTPS site on `127.0.0.1:9443`:

```bash
sudo tee /etc/nginx/sites-available/camouflage >/dev/null <<'EOF'
server {
    listen 127.0.0.1:9443 ssl http2;
    server_name vpn.example.com;

    ssl_certificate     /etc/letsencrypt/live/vpn.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vpn.example.com/privkey.pem;
    ssl_protocols TLSv1.3;

    root /var/www/html;
    index index.html;
}
EOF
sudo ln -s /etc/nginx/sites-available/camouflage /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Check (must print the page HTML):

```bash
curl -s --resolve vpn.example.com:9443:127.0.0.1 https://vpn.example.com:9443
```

---

## 4. Install 3x-ui (Xray panel) (≈10 min)

```bash
sudo -i
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

- If the installer offers to issue an SSL certificate for the panel — **decline**. The panel will only be reachable through an SSH tunnel.
- Note the printed **username, password, port, web base path**. To see them again: run `x-ui` → *View Current Settings*.

Open the panel via SSH tunnel (from your computer; `PANEL_PORT` / `PATH` from above):

```bash
ssh -L 8080:127.0.0.1:PANEL_PORT admin@SERVER_IP
# then open in browser: http://127.0.0.1:8080/PATH
```

In the panel:

1. **Panel Settings → General:** set *Listen IP* = `127.0.0.1` (panel no longer reachable from the internet at all). Change username/password. Save → restart panel. Reconnect through the tunnel.
2. **Xray version** (Dashboard → Xray → switch version): Xray ≥ 26.9.8 reportedly rejects REALITY connections from sing-box–based apps (Hiddify, Karing, NekoBox, possibly Shadowrocket) *(reported Sept 2026)*. Either stay on a version **older than 26.9.8**, or make sure the family only uses Xray-core apps (Happ, v2RayTun, v2rayNG) and keeps them updated.

---

## 5. Inbound 1 — VLESS + REALITY + Vision on 443 (main)

**Inbounds → Add Inbound:**

| Field | Value |
|---|---|
| Protocol | `vless` |
| Port | `443` |
| Transmission | `TCP (RAW)` |
| Security | `Reality` |
| uTLS | `firefox` (Chrome/Safari fingerprints reportedly flagged in 2026) |
| Target (Dest) | `127.0.0.1:9443` |
| SNI (Server Names) | `vpn.example.com` |
| Keys | click **Get New Cert** (generates private/public key) |
| Short IDs | leave auto-generated |
| Sniffing | on |

**Client** (inside the inbound):

| Field | Value |
|---|---|
| Email | person's name, e.g. `mom` |
| Flow | `xtls-rprx-vision` |
| Subscription (subId) | a random string per person, e.g. `mom-7f3k9q` — **reuse the same subId for this person in every inbound** |

Save. Open the client's share link (QR / copy) and check that the address in it is **`vpn.example.com`**, not the IP. If it shows the IP, set the inbound's *External Proxy* to `vpn.example.com` and port `443`. This is what lets you move servers later without touching the family's phones.

---

## 6. Inbound 2 — VLESS + XHTTP + REALITY on 8443 (backup)

Same as above, except:

| Field | Value |
|---|---|
| Port | `8443` |
| Transmission | `XHTTP`, mode `auto`, path e.g. `/assets` |
| Security | `Reality`, target `127.0.0.1:9443`, SNI `vpn.example.com`, uTLS `firefox`, **Get New Cert** |
| Client flow | **empty** (Vision is not used with XHTTP) |
| Client subId | same as in inbound 1 for the same person |

---

## 7. Subscription links (one link per person)

**Panel Settings → Subscription:**

| Field | Value |
|---|---|
| Enable subscription | on |
| Listen port | `2096` |
| Path | `/sub/` (or something random) |
| Domain | `vpn.example.com` |
| Certificate public key path | `/etc/letsencrypt/live/vpn.example.com/fullchain.pem` |
| Certificate private key path | `/etc/letsencrypt/live/vpn.example.com/privkey.pem` |
| Update interval | 12 (hours) |

Save, restart panel. Make x-ui reload the certificate after each automatic renewal:

```bash
echo -e '#!/bin/sh\nsystemctl restart x-ui' | sudo tee /etc/letsencrypt/renewal-hooks/deploy/x-ui.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/x-ui.sh
```

Each person's link:

```
https://vpn.example.com:2096/sub/<their subId>
```

Open it in your browser — it must return a (base64) list with both inbounds. Send each person **only their own link**, via a private channel, never posted publicly.

---

## 8. (Optional) Hysteria2 on UDP 47000

Only if the family's Internet throttles TCP. Check that their app supports Hysteria2 before bothering (Happ reportedly does *(unverified)*; v2RayTun/v2rayNG are Xray-based and may not).

```bash
bash <(curl -fsSL https://get.hy2.sh/)
```

Copy the certificate to a place the `hysteria` service user can read, and keep it updated on renewal:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/hysteria.sh >/dev/null <<'EOF'
#!/bin/sh
install -o hysteria -g hysteria -m 600 /etc/letsencrypt/live/vpn.example.com/fullchain.pem /etc/hysteria/cert.pem
install -o hysteria -g hysteria -m 600 /etc/letsencrypt/live/vpn.example.com/privkey.pem  /etc/hysteria/key.pem
systemctl restart hysteria-server
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/hysteria.sh
sudo /etc/letsencrypt/renewal-hooks/deploy/hysteria.sh
```

Config (`openssl rand -hex 16` twice for the passwords):

```bash
sudo tee /etc/hysteria/config.yaml >/dev/null <<'EOF'
listen: :47000

tls:
  cert: /etc/hysteria/cert.pem
  key: /etc/hysteria/key.pem

auth:
  type: password
  password: CHANGE_ME_AUTH_PASSWORD

obfs:
  type: salamander
  salamander:
    password: CHANGE_ME_OBFS_PASSWORD
EOF
sudo systemctl enable --now hysteria-server
sudo systemctl status hysteria-server --no-pager
```

Client link (add manually in the app, it is not part of the 3x-ui subscription):

```
hysteria2://CHANGE_ME_AUTH_PASSWORD@vpn.example.com:47000/?sni=vpn.example.com&obfs=salamander&obfs-password=CHANGE_ME_OBFS_PASSWORD#Backup-UDP
```

---

## 9. (Optional) MTProto proxy for Telegram on 2443

Expectation: works mainly on home Internet; on Russian mobile networks MTProto is often blocked regardless *(reported 2026)*. Telegram through the VPN is the primary route. All family members must keep Telegram **updated** — the April 2026 detection was fixed partly on the client side.

```bash
curl -fsSL https://get.docker.com | sudo sh

# Fake-TLS secret bound to your own domain
SECRET=$(sudo docker run --rm nineseconds/mtg:2 generate-secret --hex vpn.example.com)

sudo tee /etc/mtg.toml >/dev/null <<EOF
secret = "$SECRET"
bind-to = "0.0.0.0:3128"
EOF

sudo docker run -d --name mtg --restart unless-stopped \
  -v /etc/mtg.toml:/config.toml -p 2443:3128 nineseconds/mtg:2

# Print the tg:// link to share
sudo docker exec mtg /mtg access /config.toml
```

Check the image is mtg **≥ 2.2.8** (released 7 Apr 2026): `sudo docker image inspect nineseconds/mtg:2 --format '{{.Created}}'` must show a later date. Update later with `sudo docker pull nineseconds/mtg:2` and re-create the container.

(Note: Docker-published ports bypass ufw — fine here, since 2443 is meant to be open.)

---

## 10. Family devices

Install apps **now**, while available — several were removed from the Russian App Store in 2026, already-installed apps keep working.

| Platform | App | Where *(status Sept 2026, reported)* |
|---|---|---|
| Android | Happ or v2RayTun | Google Play |
| Android | v2rayNG | APK from GitHub |
| iOS | Happ Lite | Russian App Store (reportedly back since 11 Sep 2026) |
| iOS | Happ / v2RayTun | Non-Russian Apple ID (e.g. Kazakhstan) |

Setup per device:

1. App → **+** → *Import subscription from clipboard / QR* → paste the person's subscription link.
2. **Routing (split tunnel):** Russian sites go direct, everything else via the server. In Happ/v2RayTun choose the built-in *Russia* routing preset if present, or set rules:
   - Direct: `geosite:category-ru`, `geoip:ru`, `geoip:private`
   - Proxy: everything else
3. **Android — Russian bank / marketplace / government apps:** many detect a running VPN on the device even with split tunneling *(reported 2026)*. Put them into a separate **Work Profile** (apps: *Island* or *Shelter*) where the VPN is not active, or exclude them via the app's *per-app proxy* setting, or simply switch VPN off when using them.
4. Test in this order and note which profile works: home Wi-Fi → each mobile operator used by the family.

### Short instruction for family (RU)

> 1. Установить приложение **Happ** (Android — Google Play; iPhone — Happ Lite в App Store).
> 2. Нажать «+» → «Вставить из буфера» → вставить мою ссылку.
> 3. Включить. Если не работает — выбрать в списке другой сервер (второй — запасной).
> 4. Банки, Госуслуги, Ozon, WB — открывать с **выключенным** VPN.
> 5. Ссылку никому не пересылать.

---

## 11. Backups (do once now, then after every change)

```bash
# on the server
sudo cp /etc/x-ui/x-ui.db /home/admin/x-ui.db && sudo chown admin /home/admin/x-ui.db
# on your computer
scp admin@SERVER_IP:/home/admin/x-ui.db ./x-ui-backup-$(date +%F).db
```

Also keep copies of `/etc/hysteria/config.yaml` and `/etc/mtg.toml` if you use them. The DB contains all users and REALITY keys — **with it, existing family links keep working on a new server.**

---

## 12. If the server gets blocked (≈1 hour)

1. Rent a new VPS (preferably a different provider — blocks in 2026 often hit whole subnets).
2. Change DNS `A` record to the new IP. Wait ~5 min (TTL 300) — needed before the certificate step.
3. Repeat steps 2–4 on the new server.
4. In the new panel: restore the DB (*Panel Settings → Backup & Restore*, or `sudo x-ui stop`, copy `x-ui.db` to `/etc/x-ui/`, `sudo x-ui start`).
5. Re-do step 7's deploy hook and optional steps 8–9 if used.
6. Family does nothing: apps reconnect to `vpn.example.com`, subscription refreshes automatically. The MTProto `tg://` link contains the domain as well — no change needed.

---

## 13. Maintenance

| When | What |
|---|---|
| Monthly | `sudo apt update && sudo apt -y upgrade`; `x-ui` → update panel; check Xray version note (step 4); backup DB |
| After blocking waves / complaints | Ask family which profile fails; check from Russian nodes at check-host.net (TCP check `vpn.example.com:443`) |
| Every 60–90 days | Nothing — certbot renews automatically (`sudo certbot renew --dry-run` to verify once) |

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Connects, then stalls after a few seconds | Freeze on foreign datacenter IP | Try inbound 2 (XHTTP) or Hysteria2; if all fail on all networks → IP/subnet blocked → step 12 with another provider |
| REALITY fails only on some apps | Xray ≥ 26.9.8 vs sing-box–based client | Downgrade Xray in panel or switch to Happ/v2RayTun |
| Works on Wi-Fi, not on mobile | Operator throttling / whitelist mode | Try other profiles; in whitelist mode nothing foreign works — use Wi-Fi |
| Bank / marketplace app refuses | App detects VPN | Work Profile or VPN off for that app |
| `curl` in step 3 fails | nginx config / cert path | `sudo nginx -t`, check `/etc/letsencrypt/live/vpn.example.com/` |
| Subscription link doesn't open | Port 2096 closed or cert paths wrong | `sudo ufw status`, re-check step 7 |
| Telegram proxy "connecting…" forever | MTProto blocked on that network | Use Telegram through the VPN instead |

---

## 15. Limits (be honest with the family)

- **Mobile whitelist mode:** nothing on a foreign server works. Only Wi-Fi.
- **Blocking waves** happen roughly monthly in 2026; expect occasional 1-hour fixes (step 12).
- If direct connections keep failing on the family's provider for weeks, the next step is a Russian relay VPS (~150 ₽/mo), with the legal/identity trade-offs discussed earlier.

## Cost *(estimate)*

| Item | Cost |
|---|---|
| VPS | €4–7 / month |
| Domain | €1–10 / year |
| **Total** | **≈ €5–8 / month** |

Setup time: ~2–3 hours for a first install *(estimate)*.
