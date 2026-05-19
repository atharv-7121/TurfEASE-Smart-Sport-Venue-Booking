# Turfease Connect — Production Stack

Recommended **ready platform** for this project:

| Layer | Platform | Why |
|-------|----------|-----|
| **Frontend** | [Vercel](https://vercel.com) | Best Vite/React support, preview URLs, env UI |
| **Backend + Socket.io** | [Railway](https://railway.app) | One Node process for REST + WebSockets, simple env vars |
| **Database** | [MongoDB Atlas](https://www.mongodb.com/atlas) | Managed MongoDB, free M0 tier |
| **Payments / Auth secrets** | Railway env vars (server) + Vercel env vars (public Razorpay key only) |

**Socket.io:** Run on the **same Node server** as Express (see `server/src/index.js`). No separate real-time service unless you outgrow a single instance.

**Skip for now:** AWS ECS (more ops), separate Socket service (extra cost/complexity), Netlify (Vercel is equivalent; `netlify.toml` included if you prefer it).

---

## 1. MongoDB Atlas

1. Create a free **M0** cluster.
2. Database Access → create user + password.
3. Network Access → allow `0.0.0.0/0` (or Railway’s egress IPs in production).
4. Connect → copy **connection string** → set as `MONGODB_URI` on Railway.

---

## 2. Backend (Railway)

1. Push repo to GitHub.
2. Railway → **New Project** → **Deploy from GitHub** → select repo.
3. Settings → set **Root Directory** to repo root (uses `railway.toml`) **or** set service root to `server` and start command `npm start`.
4. **Variables** (Settings → Variables):

   | Variable | Notes |
   |----------|--------|
   | `MONGODB_URI` | Atlas connection string |
   | `JWT_SECRET` | Long random string (32+ chars) |
   | `JWT_EXPIRES_IN` | e.g. `7d` |
   | `RAZORPAY_KEY_ID` | From Razorpay dashboard |
   | `RAZORPAY_KEY_SECRET` | **Server only** — never in frontend |
   | `CLIENT_ORIGIN` | `https://your-app.vercel.app` |
   | `PORT` | Railway sets this automatically; optional `3001` locally |

5. Deploy → copy public URL (e.g. `https://turfease-api-production.up.railway.app`).
6. Verify: `GET https://<api-url>/health`

**Render alternative:** Connect repo → **New Blueprint** → use root `render.yaml`, then add secret env vars in the dashboard.

---

## 3. Frontend (Vercel)

1. [vercel.com](https://vercel.com) → Import Git repo.
2. Framework: **Vite** (auto-detected).
3. Build: `npm run build` · Output: `dist` (see `vercel.json`).
4. **Environment variables:**

   | Variable | Value |
   |----------|--------|
   | `VITE_API_URL` | Railway API URL |
   | `VITE_SOCKET_URL` | Same as API URL (Socket.io on same host) |
   | `VITE_RAZORPAY_KEY_ID` | Public key only (`rzp_test_` / `rzp_live_`) |

5. Deploy → set Railway `CLIENT_ORIGIN` to your Vercel URL and redeploy API if CORS fails.

---

## 4. Razorpay & JWT — secret rules

| Secret | Where |
|--------|--------|
| `RAZORPAY_KEY_SECRET` | Railway / Render only |
| `JWT_SECRET` | Railway / Render only |
| `RAZORPAY_KEY_ID` | Server env + `VITE_RAZORPAY_KEY_ID` on Vercel (public) |
| `MONGODB_URI` | Server only |

Never commit `.env` files. Use `.env.example` as a checklist.

---

## 5. Local development

```bash
# Terminal 1 — API + sockets
cd server
cp .env.example .env   # fill values
npm install
npm run dev

# Terminal 2 — frontend
cp .env.example .env
npm install
npm run dev
```

Frontend: `http://localhost:8080` · API: `http://localhost:3001`

---

## 6. When to change platforms

| Need | Move to |
|------|---------|
| More control, containers, scale | AWS ECS + ALB (enable sticky sessions for Socket.io) |
| High Socket.io traffic | Same server first; then Redis adapter + horizontal scale |
| Prefer Netlify for UI | Use `netlify.toml`; keep Railway for API |

---

## Quick checklist

- [ ] Atlas cluster + `MONGODB_URI`
- [ ] Railway service live + `/health` OK
- [ ] Vercel deploy + `VITE_*` env vars
- [ ] `CLIENT_ORIGIN` matches Vercel URL
- [ ] Razorpay test keys in server; live keys only in production
