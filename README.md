# lumino-server-website

Landing page for the LuminoServer Minecraft server. It's a single static
`index.html` file (CSS/JS embedded, no build step, no dependencies).

## Run it locally

Pick whichever you already have installed — any of these work, no `npm install` needed:

**Node.js** (uses the included zero-dependency server):
```bash
node server.js        # http://localhost:8080
# or: npm start
# custom port:
PORT=3000 node server.js
```

**Python 3:**
```bash
python3 -m http.server 8080   # http://localhost:8080
```

**npx (no local install required):**
```bash
npx serve .
```

Then open the printed URL in your browser.

## Editing content

Placeholder connection addresses live in `index.html` (search for
`example.com`) — update them with your real playit.gg Java/Bedrock
addresses before deploying.

## Deploying

Since it's a static file, any static host works (Nginx, Apache, Caddy,
GitHub Pages, Netlify, Vercel, etc.) — just serve `index.html` from the
repo root. `server.js` is meant for local/simple hosting; for production
behind a reverse proxy, point Nginx/Caddy directly at the file instead.

## Deploying on a fresh Ubuntu server (step by step)

SSH into the server, then:

**1. Get the code**
```bash
sudo apt update && sudo apt install -y git nginx
sudo git clone https://github.com/palkipl/lumino-server-website.git /var/www/lumino-server-website
```

**2. Edit the placeholder IPs** (optional, but you'll want your real ones)
```bash
sudo nano /var/www/lumino-server-website/index.html
# find/replace the "example.com" addresses, save with Ctrl+O, exit with Ctrl+X
```

**3. Point Nginx at it**
```bash
sudo cp /var/www/lumino-server-website/deploy/nginx.conf /etc/nginx/sites-available/lumino-server-website
sudo ln -s /etc/nginx/sites-available/lumino-server-website /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default   # avoid the default page conflicting
sudo nginx -t                                  # should say "syntax is ok"
sudo systemctl reload nginx
```

**4. Open the firewall** (only if `ufw` is active — check with `sudo ufw status`)
```bash
sudo ufw allow 'Nginx Full'
```

**5. Visit it**
```
http://your_server_ip
```
That's it — Nginx serves the file directly, starts on boot, and needs no Node/Python process running.

**To pull future updates:**
```bash
cd /var/www/lumino-server-website
sudo git pull
```

### Alternative: run it with the Node server instead of Nginx

Only do this if you'd rather not use Nginx. Requires Node.js (`sudo apt install -y nodejs`).
```bash
sudo cp /var/www/lumino-server-website/deploy/luminoserver-website.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now luminoserver-website
sudo ufw allow 8080/tcp   # if ufw is active
```
Then visit `http://your_server_ip:8080`. Check status/logs with
`sudo systemctl status luminoserver-website` or `sudo journalctl -u luminoserver-website -f`.

## Custom domain via Cloudflare Tunnel (cloudflared)

Use this if you want the site reachable at a real domain (e.g.
`luminoserver.com`) without opening any inbound ports — useful when the
server is behind NAT or reachable only through a tunnel like playit.gg.
Cloudflare also terminates HTTPS for you automatically.

These steps create a **second, independent tunnel** dedicated to this site —
distinct tunnel name, config file, and systemd service — so an existing
`cloudflared` tunnel already running on the same server for another project
is untouched. Skip the install step if `cloudflared` is already on the
server.

**Prerequisites (done once, in your browser/registrar — not on the server):**
1. Own the domain and add it as a site in the
   [Cloudflare dashboard](https://dash.cloudflare.com) (free plan works).
2. Point the domain's nameservers at the two Cloudflare gave you, at your
   registrar. Wait until Cloudflare shows the zone as **Active**.

**On the server:**

1. Install `cloudflared` (skip if you already have it from another project):
   ```bash
   sudo mkdir -p /usr/share/keyrings
   curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
   echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
   sudo apt update && sudo apt install -y cloudflared
   ```

2. Authenticate as root, since the systemd service in step 6 runs as root
   and needs to read the credentials from `/root/.cloudflared/` — running
   as your normal user instead puts them in the wrong home directory and
   the service will fail with "credentials file doesn't exist". Skip this
   if `/root/.cloudflared/cert.pem` already exists from your other project
   (opens a link — open it in any browser, sign in, and pick the
   `luminoserver.com` zone to authorize). If the cert already exists but
   was authorized for a different zone, re-run this and pick
   `luminoserver.com` this time:
   ```bash
   sudo cloudflared tunnel login
   ```

3. Create a **new, separately-named tunnel** for this site (also as root,
   for the same reason — keeps credentials under `/root/.cloudflared/`):
   ```bash
   sudo cloudflared tunnel create luminoserver-website
   ```
   Note the tunnel ID it prints — you'll need it next.

4. Set up a **dedicated config file** (a distinct path, not
   `/etc/cloudflared/config.yml`, so it can't overwrite another project's
   config):
   ```bash
   sudo mkdir -p /etc/cloudflared
   sudo cp /var/www/lumino-server-website/deploy/cloudflared-luminoserver-config.yml /etc/cloudflared/luminoserver-config.yml
   sudo nano /etc/cloudflared/luminoserver-config.yml   # replace <TUNNEL_ID> in both places
   ```

5. Route the domain to this tunnel (creates the DNS records for you; use
   `sudo` here too so it reads the same root-owned cert/tunnel list from
   steps 2-3):
   ```bash
   sudo cloudflared tunnel route dns luminoserver-website luminoserver.com
   sudo cloudflared tunnel route dns luminoserver-website www.luminoserver.com
   ```

6. Install a **dedicated systemd service** (separate unit name — `cloudflared-luminoserver`
   — so it starts/stops independently of any existing `cloudflared` service):
   ```bash
   sudo cp /var/www/lumino-server-website/deploy/cloudflared-luminoserver.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable --now cloudflared-luminoserver
   ```

Then visit `https://luminoserver.com` — Cloudflare handles TLS, and traffic
is tunneled straight to Nginx on the server with no open inbound ports
needed, running alongside your other tunnel with no shared state. Check
status/logs with `sudo systemctl status cloudflared-luminoserver` or
`sudo journalctl -u cloudflared-luminoserver -f`.
