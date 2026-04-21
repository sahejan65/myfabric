# MyFabric — Complete Deployment Guide

## 📁 Project Structure

```
myfabric/              ← React + Tailwind frontend (Vite)
myfabric-backend/      ← Node.js + Express + MongoDB backend
```

---

## ⚡ LOCAL DEVELOPMENT (Run on your laptop)

### Prerequisites
- Node.js v18+ — https://nodejs.org
- MongoDB Community — https://www.mongodb.com/try/download/community
- Git (optional)

### Step 1 — Start MongoDB
```bash
# macOS (Homebrew)
brew services start mongodb-community

# Windows — start from Services or run:
net start MongoDB

# Linux
sudo systemctl start mongod
```

### Step 2 — Backend Setup
```bash
cd myfabric-backend

# Install dependencies
npm install

# Create environment file
cp .env.example .env

# Edit .env — set your values:
#   MONGO_URI=mongodb://localhost:27017/myfabric
#   JWT_SECRET=any_long_random_string_here_min32chars
#   FRONTEND_URL=http://localhost:5173
#   PORT=5000

# Seed database with demo data (products + users + stitching)
npm run seed

# Start backend server
npm run dev
# → API running at http://localhost:5000
```

### Step 3 — Frontend Setup
```bash
cd myfabric

# Install dependencies
npm install

# Create environment file
cp .env.example .env.local

# Edit .env.local:
#   VITE_API_URL=http://localhost:5000/api

# Start frontend
npm run dev
# → App running at http://localhost:5173
```

### Demo Login Credentials
| Role    | Email                    | Password     |
|---------|--------------------------|--------------|
| Admin   | admin@myfabric.in        | admin123     |
| Manager | manager@myfabric.in      | manager123   |
| User    | user@myfabric.in         | user123      |

---

## 🚀 PRODUCTION DEPLOYMENT

### Option A — Vercel (Frontend) + Render (Backend) + MongoDB Atlas
**Recommended — All free tiers available**

---

#### STEP 1 — MongoDB Atlas (Free Cloud Database)

1. Go to https://www.mongodb.com/atlas
2. Create free account → click **"Build a Database"**
3. Choose **Free tier (M0)** → Select region **Mumbai (ap-south-1)**
4. Create username + password (save these!)
5. In **Network Access** → click **"Add IP Address"** → **"Allow Access from Anywhere"** (0.0.0.0/0)
6. In **Database** → click **"Connect"** → **"Drivers"** → copy the connection string:
   ```
   mongodb+srv://USERNAME:PASSWORD@cluster0.xxxxx.mongodb.net/myfabric?retryWrites=true&w=majority
   ```
   Replace `USERNAME` and `PASSWORD` with what you created.

---

#### STEP 2 — Deploy Backend on Render.com (Free)

1. Go to https://render.com → Sign up with GitHub
2. Click **"New +"** → **"Web Service"**
3. Connect your GitHub repo (push `myfabric-backend/` folder to GitHub first)
   - Or use **"Deploy from existing repo"**
4. Settings:
   - **Name:** `myfabric-api`
   - **Root Directory:** `.` (or `myfabric-backend` if in monorepo)
   - **Runtime:** `Node`
   - **Build Command:** `npm install`
   - **Start Command:** `node src/server.js`
   - **Region:** Singapore (closest to India)
5. Click **"Advanced"** → **"Add Environment Variables"**:
   ```
   NODE_ENV        = production
   PORT            = 5000
   MONGO_URI       = mongodb+srv://user:pass@cluster.mongodb.net/myfabric
   JWT_SECRET      = your_super_secret_key_minimum_32_characters_long
   FRONTEND_URL    = https://your-app.vercel.app   (fill after Step 3)
   ```
6. Click **"Create Web Service"** → wait 3-5 minutes
7. Copy your backend URL: `https://myfabric-api.onrender.com`
8. **Seed the database** (one time):
   - In Render dashboard → **Shell** tab → run:
   ```bash
   node src/seed.js
   ```

---

#### STEP 3 — Deploy Frontend on Vercel (Free)

1. Go to https://vercel.com → Sign up with GitHub
2. Click **"New Project"** → import your `myfabric/` folder
3. Settings:
   - **Framework Preset:** Vite
   - **Root Directory:** `myfabric` (if in monorepo) or `.`
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`
4. Click **"Environment Variables"** → add:
   ```
   VITE_API_URL = https://myfabric-api.onrender.com/api
   ```
5. Click **"Deploy"** → wait 2 minutes
6. Copy your frontend URL: `https://myfabric.vercel.app`
7. **Go back to Render** → update `FRONTEND_URL` to your Vercel URL → **Redeploy**

✅ Done! Your app is live.

---

### Option B — VPS / DigitalOcean Droplet

#### STEP 1 — Create Droplet
1. https://digitalocean.com → Create Droplet
2. Choose: Ubuntu 22.04 · Basic · $6/month (1GB RAM)
3. Region: Bangalore

#### STEP 2 — Setup Server
```bash
# SSH into your server
ssh root@YOUR_SERVER_IP

# Update system
apt update && apt upgrade -y

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Install MongoDB
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | apt-key add -
echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-7.0.list
apt update && apt install -y mongodb-org
systemctl start mongod && systemctl enable mongod

# Install PM2 (process manager keeps app running)
npm install -g pm2

# Install Nginx (web server)
apt install -y nginx
```

#### STEP 3 — Deploy Code
```bash
# Upload your code (from your laptop)
scp -r myfabric-backend/ root@YOUR_IP:/var/www/myfabric-backend
scp -r myfabric/dist/    root@YOUR_IP:/var/www/myfabric-frontend

# On server — setup backend
cd /var/www/myfabric-backend
npm install --production

# Create .env file
nano .env
# Paste and fill:
# PORT=5000
# MONGO_URI=mongodb://localhost:27017/myfabric
# JWT_SECRET=your_secret_here
# FRONTEND_URL=http://YOUR_DOMAIN.com
# NODE_ENV=production

# Seed database
node src/seed.js

# Start with PM2
pm2 start src/server.js --name myfabric-api
pm2 save && pm2 startup
```

#### STEP 4 — Configure Nginx
```bash
nano /etc/nginx/sites-available/myfabric
```
Paste:
```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN.com;

    # Frontend (React SPA)
    location / {
        root /var/www/myfabric-frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # Backend API
    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Uploaded images
    location /uploads {
        proxy_pass http://localhost:5000;
    }
}
```
```bash
ln -s /etc/nginx/sites-available/myfabric /etc/nginx/sites-enabled/
nginx -t && systemctl restart nginx
```

#### STEP 5 — Free SSL Certificate
```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d YOUR_DOMAIN.com
# Follow prompts — auto-renews every 90 days
```

---

### Option C — Railway.app (Easiest, one-click)

1. Go to https://railway.app
2. Click **"New Project"** → **"Deploy from GitHub"**
3. Select your backend repo
4. Add environment variables in dashboard (same as Render above)
5. Railway auto-detects Node.js and deploys
6. Add a MongoDB plugin: **"New"** → **"Database"** → **"MongoDB"**
7. Copy `MONGO_URL` from MongoDB plugin into your service env vars
8. Run seed: **"Shell"** → `node src/seed.js`

---

## 🔄 After Deployment — Update Frontend API URL

Once your backend is live, update `VITE_API_URL` in your frontend:

```bash
# In myfabric/.env.local (local dev)
VITE_API_URL=http://localhost:5000/api

# In Vercel environment variables (production)
VITE_API_URL=https://your-backend.onrender.com/api
```

Then rebuild: `npm run build`

---

## 🔑 Environment Variables Reference

### Frontend (myfabric/.env.local)
| Variable        | Example                                    | Required |
|-----------------|--------------------------------------------|----------|
| VITE_API_URL    | https://myfabric-api.onrender.com/api      | ✅ Yes   |

### Backend (myfabric-backend/.env)
| Variable        | Example                                    | Required |
|-----------------|--------------------------------------------|----------|
| PORT            | 5000                                       | ✅ Yes   |
| MONGO_URI       | mongodb+srv://user:pass@cluster/myfabric   | ✅ Yes   |
| JWT_SECRET      | any_random_string_min_32_chars             | ✅ Yes   |
| FRONTEND_URL    | https://myfabric.vercel.app                | ✅ Yes   |
| NODE_ENV        | production                                 | ✅ Yes   |

---

## 📦 Useful Commands

```bash
# Backend
npm run dev          # Start dev server with auto-reload
npm start            # Start production server
npm run seed         # Seed database with demo data

# Frontend
npm run dev          # Start dev server (http://localhost:5173)
npm run build        # Build for production (output: dist/)
npm run preview      # Preview production build locally

# PM2 (VPS only)
pm2 list             # See running processes
pm2 logs myfabric-api  # View backend logs
pm2 restart myfabric-api  # Restart backend
pm2 stop myfabric-api    # Stop backend
```

---

## 🐛 Troubleshooting

| Problem | Fix |
|---------|-----|
| `CORS error` | Update `FRONTEND_URL` in backend .env to match your actual frontend URL |
| `Cannot connect to MongoDB` | Check MONGO_URI is correct, IP whitelisted in Atlas |
| `JWT errors` | Make sure `JWT_SECRET` is same in .env and not changed after deploy |
| `Products not loading` | Run `npm run seed` to populate database |
| `White screen on refresh` | Add `vercel.json` rewrites (already included) or Nginx `try_files` |
| `Images not uploading` | Create `uploads/` folder in backend: `mkdir -p uploads` |
| `Build fails on Vercel` | Ensure `VITE_API_URL` env var is set in Vercel dashboard |
