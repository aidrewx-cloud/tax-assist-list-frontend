# Tax Assist List — Go-Live Deploy Guide

## What You Need

| Item | Cost | Purpose |
|------|------|---------|
| VPS (DigitalOcean, Railway, Render) | ~$5-12/mo | Hosts backend + frontend |
| Domain name | ~$10-15/yr | Your URL (e.g. taxassistlist.com) |
| MongoDB Atlas (free tier) | Free | Database |
| SSL Certificate | Free | HTTPS (via Let's Encrypt) |

---

## Step-by-Step

### 1. Install Node.js (on your Mac first to test locally)

```bash
# Install via Homebrew
brew install node

# Verify
node --version   # Should show v18+
npm --version
```

### 2. Set Up MongoDB Atlas (Free)

1. Go to https://cloud.mongodb.com and create a free account
2. Create a free M0 cluster (choose closest region)
3. Under "Database Access" → Add a database user (save the username/password)
4. Under "Network Access" → Add your IP (or 0.0.0.0/0 for anywhere)
5. Click "Connect" → "Connect your application" → Copy the connection string
6. It looks like: `mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/tax_assist_list`

### 3. Configure Environment Variables

Edit `/backend/.env` with your production values:

```env
PORT=5000
NODE_ENV=production
MONGODB_URI=mongodb+srv://YOUR_USER:YOUR_PASS@cluster0.xxxxx.mongodb.net/tax_assist_list
JWT_SECRET=generate-a-long-random-string-here-64-chars-minimum
JWT_EXPIRES_IN=7d
ADMIN_USERNAME=your_admin_username
ADMIN_PASSWORD=your_secure_password
CORS_ORIGIN=https://yourdomain.com
IRSLOGICS_API_ENDPOINT=https://api.irslogics.com/v1/leads
IRSLOGICS_API_KEY=your_key_from_irslogics
IRSLOGICS_COMPANY_ID=your_company_id
```

To generate a secure JWT secret:
```bash
openssl rand -hex 32
```

### 4. Test Locally

```bash
# Navigate to backend
cd "/Users/claudex/Documents/Tax Company Code/backend"

# Install dependencies (already done but just in case)
npm install

# Start the server
node server.js

# You should see:
# [MongoDB] Connected
# [Server] API available at: http://localhost:5000/api
```

Then open `index8.html` in your browser — the lead form should now submit successfully.

Open `portal.html` — log in with your ADMIN_USERNAME/ADMIN_PASSWORD from .env.

### 5. Create Your First Admin User (in MongoDB)

Once the backend is running locally, create a proper admin user:

```bash
# In a new terminal, run this one-liner
node -e "
const mongoose = require('mongoose');
const User = require('./models/User');
require('dotenv').config();
mongoose.connect(process.env.MONGODB_URI).then(async () => {
  await User.create({
    username: 'youradmin',
    password: 'YourSecurePass123',
    email: 'you@email.com',
    role: 'admin'
  });
  console.log('Admin user created!');
  process.exit(0);
}).catch(e => { console.error(e); process.exit(1); });
"
```

### 6. Choose a Hosting Provider

**Option A: DigitalOcean Droplet ($6/mo) — Recommended**

```bash
# SSH into your droplet
ssh root@your-droplet-ip

# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 (keeps your app running)
sudo npm install -g pm2

# Clone or upload your project
# Option 1: Git
git clone your-repo-url /var/www/taxassistlist
# Option 2: SCP from your Mac
scp -r "/Users/claudex/Documents/Tax Company Code/backend" root@your-ip:/var/www/taxassistlist/backend

# Set up the backend
cd /var/www/taxassistlist/backend
npm install --production
cp .env.example .env
nano .env  # Fill in production values

# Start with PM2
pm2 start server.js --name "tax-api"
pm2 save
pm2 startup  # Auto-restart on reboot
```

**Option B: Railway (easiest, $5/mo)**

1. Push your `backend/` folder to a GitHub repo
2. Go to https://railway.app → New Project → Deploy from GitHub
3. Add your .env variables in the Railway dashboard
4. Railway gives you a URL like `tax-api.up.railway.app`

**Option C: Render (free tier available)**

1. Push to GitHub
2. Go to https://render.com → New Web Service
3. Point to your repo, set root to `backend/`
4. Add environment variables
5. Deploy

### 7. Set Up Your Domain

1. Buy a domain (Namecheap, GoDaddy, Cloudflare, etc.)
2. Point DNS to your server:
   - **A Record**: `@` → your server IP
   - **A Record**: `www` → your server IP
   - Or if using Railway/Render: **CNAME** → their provided URL

### 8. Set Up SSL (HTTPS)

**On DigitalOcean/VPS with Nginx:**

```bash
# Install Nginx
sudo apt install nginx

# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Create Nginx config
sudo nano /etc/nginx/sites-available/taxassistlist
```

Paste this Nginx config:

```nginx
server {
    server_name yourdomain.com www.yourdomain.com;

    # Serve frontend files
    root /var/www/taxassistlist/frontend;
    index index8.html;

    # API proxy
    location /api/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    location / {
        try_files $uri $uri/ /index8.html;
    }
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/taxassistlist /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# Get SSL certificate
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renew (certbot sets this up automatically)
sudo certbot renew --dry-run
```

**On Railway/Render:** SSL is automatic — nothing to do.

### 9. Upload Frontend Files

```bash
# On your VPS, create frontend directory
mkdir -p /var/www/taxassistlist/frontend

# From your Mac, upload the files
scp "/Users/claudex/Documents/Tax Company Code/index8.html" root@your-ip:/var/www/taxassistlist/frontend/index.html
scp "/Users/claudex/Documents/Tax Company Code/portal.html" root@your-ip:/var/www/taxassistlist/frontend/portal.html
```

### 10. Update Frontend URLs

Before uploading, update these in `index8.html`:
- Replace placeholder phone numbers with real ones:
  - Alleviate: `(800) 555-1234` → your real number
  - Optima: `(800) 555-5678` → your real number
  - Anthem: `(800) 555-9012` → your real number
- The API_BASE auto-detects same-origin in production, so no change needed

In `portal.html`:
- On first login, go to Settings → set the API URL to your production backend URL
- Or it auto-detects if served from the same domain

### 11. Final Checklist

- [ ] MongoDB Atlas cluster running with your data
- [ ] Backend server running (PM2 or Railway/Render)
- [ ] Frontend files uploaded and accessible
- [ ] Domain pointing to server
- [ ] SSL certificate installed (HTTPS working)
- [ ] Real phone numbers in landing page
- [ ] Admin user created in database
- [ ] .env has production JWT_SECRET (not the default)
- [ ] IRSLogics API key configured (if using)
- [ ] Test a lead submission end-to-end
- [ ] Test portal login and lead viewing
- [ ] Test campaign creation and spend logging

---

## File Map

```
Tax Company Code/
├── index8.html          ← Landing page (serve as index.html)
├── portal.html          ← Admin portal (private URL)
├── admin.html           ← Simple CRM view (legacy, replaced by portal)
├── backend/
│   ├── server.js        ← Express API server
│   ├── .env             ← Environment config
│   ├── models/
│   │   ├── Lead.js      ← Lead data model
│   │   ├── User.js      ← Admin user model
│   │   └── Campaign.js  ← Campaign/ad spend model
│   ├── routes/
│   │   ├── leads.js     ← Lead CRUD endpoints
│   │   ├── auth.js      ← Login/auth endpoints
│   │   ├── analytics.js ← Dashboard analytics
│   │   ├── campaigns.js ← Campaign management
│   │   ├── revenue.js   ← P&L / revenue tracking
│   │   └── webhooks.js  ← IRSLogics webhook receiver
│   ├── services/
│   │   └── irslogics.js ← CRM integration
│   └── middleware/
│       └── auth.js      ← JWT auth middleware
```

## Default Login Credentials

**Username:** `admin`
**Password:** `password`

Change these immediately after first login via the portal Settings page or by updating ADMIN_USERNAME/ADMIN_PASSWORD in `.env`.
