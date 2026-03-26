# supercooled.org — Maintenance Guide

## Infrastructure Overview

| Thing | Where |
|---|---|
| Landing page | GitHub repo → GitHub Pages |
| Streamlit apps | DigitalOcean droplet `161.35.185.29` |
| DNS | Cloudflare |
| SSL | Certbot (auto-renews) |

Current apps on the droplet:

| App | Port | URL |
|---|---|---|
| heatflux | 8501 | heatflux.supercooled.org |
| historicheatflux | 8502 | historicheatflux.supercooled.org |

---

## Accessing the Droplet

SSH:
```bash
ssh root@161.35.185.29
```
Or use the DigitalOcean browser console as a fallback.

---

## Updating an Existing App

Pull latest code from GitHub and restart the service:

```bash
cd /opt/heatflux
git pull
systemctl restart heatflux
```

```bash
cd /opt/historicheatflux
git pull
systemctl restart historicheatflux
```

---

## Pushing Changes Made on the Droplet to GitHub

If you edit files directly on the droplet (e.g. requirements.txt):

```bash
cd /opt/heatflux          # or /opt/historicheatflux
git add .
git commit -m "describe what you changed"
git push
```

If git prompts for credentials, re-embed the token in the remote URL:
```bash
git remote set-url origin https://ski907:YOUR_GITHUB_TOKEN@github.com/ski907/REPONAME.git
```

---

## Updating the Landing Page

Edit `index.html` in your GitHub Pages repo and push — deploys automatically within ~1 minute.  
Hard refresh (`Ctrl+Shift+R`) if your browser shows the old version.

---

## Adding a New App

> ⚠️ Every time you see `NEWAPP` below, replace it with your actual app name (e.g. `permafrost`).  
> Every time you see `NEW_PORT`, replace it with the next unused port number (next one is `8503`).

### 1. Clone and install

```bash
git clone https://github.com/ski907/NEWAPP.git /opt/NEWAPP
cd /opt/NEWAPP
python3 -m venv venv
source venv/bin/activate
pip install setuptools
pip install -r requirements.txt
```

### 2. Create the systemd service

```bash
nano /etc/systemd/system/NEWAPP.service
```

Paste this — replacing `NEWAPP`, `NEW_PORT`, and `main.py` with your entry filename:

```ini
[Unit]
Description=NEWAPP Streamlit App
After=network.target

[Service]
User=root
WorkingDirectory=/opt/NEWAPP
Environment="PATH=/opt/NEWAPP/venv/bin"
ExecStart=/opt/NEWAPP/venv/bin/streamlit run main.py --server.port NEW_PORT --server.address 127.0.0.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable NEWAPP
systemctl start NEWAPP
systemctl status NEWAPP
```

### 3. Create the Nginx config

```bash
nano /etc/nginx/sites-available/NEWAPP
```

Paste this — replacing `NEWAPP` and `NEW_PORT`:

```nginx
server {
    listen 80;
    server_name NEWAPP.supercooled.org;

    location / {
        proxy_pass http://127.0.0.1:NEW_PORT;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/NEWAPP /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

### 4. Add DNS record in Cloudflare

| Type | Name | Target | Proxy |
|---|---|---|---|
| A | `NEWAPP` | `161.35.185.29` | DNS only (gray cloud) ☁️ |

### 5. Get SSL certificate

```bash
certbot --nginx -d NEWAPP.supercooled.org
```

### 6. Add to landing page

In your GitHub Pages `index.html`, copy this block inside the projects section and push:

```html
<a class="project" href="https://NEWAPP.supercooled.org">
  <div class="project-title">NEWAPP</div>
  <div class="project-desc">Short description here</div>
</a>
```

### 7. Update this doc

Add the new app to the table at the top of this file.

---

## Troubleshooting

**App not loading:**
```bash
systemctl status APPNAME
journalctl -u APPNAME -n 50
```

**Nginx error:**
```bash
nginx -t
systemctl status nginx
```

**SSL certificate expired:**
```bash
certbot renew
```
