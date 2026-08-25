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
