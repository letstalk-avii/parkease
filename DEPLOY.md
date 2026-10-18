# ParkEase – Deployment Guide

Two paths are covered: **Docker** (one command, anywhere with Docker) and **Managed hosting** (no server to maintain). Pick one.

## Option A — Docker (recommended for a viva demo or your own VPS)

### What's in the repo
```
server/Dockerfile      -> Node API image
client/Dockerfile      -> multi-stage: builds the React app, then serves it with Nginx
client/nginx.conf      -> serves the SPA and proxies /api/* to the server container
docker-compose.yml     -> runs mongo + server + client together
```

### Run it locally
```bash
cd parkease
cp server/.env.example server/.env     # edit JWT_SECRET; MONGO_URI is overridden by compose automatically
docker compose up --build
```
- Frontend: http://localhost
- API: http://localhost:5000/api (also reachable at http://localhost/api through Nginx)
- MongoDB data persists in the `mongo_data` Docker volume.

Seed demo data once the containers are up:
```bash
docker compose exec server npm run seed
```

Stop everything with `docker compose down` (add `-v` to also delete the database volume).

### Deploy the same setup to a VPS (DigitalOcean, AWS EC2, Hetzner, etc.)
1. Provision a small Linux VM (1–2 GB RAM is enough) and install Docker + the Compose plugin.
2. Copy the project to the server: `git clone <your repo>` or `scp` the folder.
3. Create `server/.env` on the server with a strong `JWT_SECRET` and your Razorpay/SMTP/Twilio keys if you have them.
4. `docker compose up -d --build`, then `docker compose exec server npm run seed` once.
5. Point your domain's A record at the VM's IP. For HTTPS, put Caddy or Nginx + Certbot in front of port 80, or add Cloudflare in front of the VM (easiest: Cloudflare proxy with "Flexible" SSL, or a free Caddy reverse proxy container that gets certificates automatically).
6. Restarting after a code change: `git pull && docker compose up -d --build`.

### Deploy the same containers to a container host (Render, Railway, Fly.io)
These platforms build from a Dockerfile directly, so no compose file is needed:
1. Push the repo to GitHub.
2. Create a **MongoDB Atlas** free cluster (see Option B, step 1) — easier than running Mongo in a container on these platforms.
3. Create a new "Web Service" from `server/Dockerfile`, set `MONGO_URI`, `JWT_SECRET` and the optional keys as environment variables, expose port 5000.
4. Create a second "Web Service" from `client/Dockerfile`, with build argument `VITE_API_URL` set to your server's public URL + `/api` (e.g. `https://parkease-api.onrender.com/api`), expose port 80.
5. Run the seed script once from the platform's shell/console: `npm run seed`.

## Option B — Managed hosting (no Docker, no server to manage)

| Part | Service | Why |
|---|---|---|
| Database | MongoDB Atlas (free tier) | Managed, no install |
| Backend | Render or Railway (free/low-cost web service) | Deploys a Node app from GitHub directly |
| Frontend | Vercel or Netlify (free tier) | Built for static/Vite sites, auto HTTPS |

### Steps
1. **Database:** create a free cluster at mongodb.com/atlas, add a database user, allow access from anywhere (0.0.0.0/0) for simplicity, and copy the connection string.
2. **Backend (Render):**
   - New "Web Service" → connect your GitHub repo → root directory `server`.
   - Build command: `npm install`. Start command: `node server.js`.
   - Environment variables: `MONGO_URI` (from Atlas), `JWT_SECRET`, `PORT=5000`, and optionally `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `SMTP_*`, `TWILIO_*`.
   - After the first deploy, open the Render shell and run `npm run seed` once.
   - Note the service URL, e.g. `https://parkease-api.onrender.com`.
3. **Frontend (Vercel):**
   - New project → import the repo → root directory `client`.
   - Framework preset: Vite. Build command: `npm run build`. Output directory: `dist`.
   - Environment variable: `VITE_API_URL = https://parkease-api.onrender.com/api`.
   - Deploy. Vercel gives you a URL and free HTTPS.
4. Update CORS if needed: the backend's `cors()` call currently allows all origins, which is fine for a demo; for production, restrict it to your Vercel domain in `server/server.js`.

## Checklist before showing it to anyone
- [ ] `JWT_SECRET` is a long random string, not the example value.
- [ ] Database seeded (`npm run seed`) so demo logins work.
- [ ] If using Razorpay, confirm you're on **test mode** keys unless you intend to take real payments.
- [ ] Frontend's `VITE_API_URL` points at the deployed backend, not localhost.
- [ ] HTTPS is in place if the app is public (Vercel/Render/Cloudflare all provide this automatically; a bare VPS needs Caddy/Certbot).
