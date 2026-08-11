# 🚀 BlouseHub — AWS EC2 Deployment Guide (Single Instance)

> **Stack**: Django (Backend) + Next.js (Frontend) + Nginx + GitHub Actions CI/CD
>
> **Goal**: Host everything on ONE AWS EC2 server with automated deployments via **SSH** on every `git push`

---

## 📐 Architecture

```
                    ┌─────────────────────────┐
                    │       INTERNET           │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     YOUR DOMAIN / IP     │
                    │   (e.g. blousehub.com)   │
                    └────────────┬────────────┘
                                 │
            ┌────────────────────▼────────────────────┐
            │          AWS EC2 INSTANCE                │
            │          (Ubuntu 24.04 LTS)              │
            │                                          │
            │   ┌──────────────────────────────────┐   │
            │   │         NGINX (Port 80/443)       │   │
            │   │      Acts as Reverse Proxy        │   │
            │   └───────┬──────────────┬───────────┘   │
            │           │              │               │
            │    ┌──────▼──────┐ ┌─────▼──────────┐   │
            │    │  Next.js    │ │  Django +       │   │
            │    │  (PM2)      │ │  Gunicorn       │   │
            │    │  Port 3000  │ │  Port 8000      │   │
            │    │             │ │                  │   │
            │    │  Handles:   │ │  Handles:        │   │
            │    │  /          │ │  /api/           │   │
            │    │  All pages  │ │  /admin/         │   │
            │    └─────────────┘ │  /static/        │   │
            │                    │  /media/         │   │
            │                    └─────────────────┘   │
            └──────────────────────────────────────────┘
```

**How the CI/CD works (SSH Deployment):**
```
┌──────────┐     ┌──────────────────┐     ┌──────────────────────────────────┐
│  GitHub   │────▶│  GitHub Actions   │────▶│  EC2 (via SSH)                  │
│  (push)   │     │  (ssh-action)     │     │  git pull → build → restart    │
└──────────┘     └──────────────────┘     └──────────────────────────────────┘
```
1. You push code to `master` on GitHub
2. GitHub Actions **SSHs into your EC2** using your `.pem` key
3. On EC2, it runs: **git pull → install deps → build → restart services**
4. Your site is updated! 🎉

> 💡 **Why SSH?** — No S3, no CodeDeploy, no IAM roles, no agent installation. Just your SSH key as a GitHub Secret. Simple, fast, and free.

---

---

# STEP 1: Create an AWS Account

> ⏱️ Time: 5-10 minutes

If you already have an AWS account, skip to Step 2.

1. Go to [https://aws.amazon.com](https://aws.amazon.com)
2. Click **"Create an AWS Account"**
3. Enter your email, password, and account name
4. Choose **"Personal"** account type
5. Enter your credit/debit card details (you won't be charged if you stay in free tier)
6. Complete phone verification
7. Select the **"Basic Support - Free"** plan
8. Sign in to the **AWS Management Console**

---

---

# STEP 2: Launch an EC2 Instance

> ⏱️ Time: 10 minutes

## 2.1 — Navigate to EC2

1. Sign in to [AWS Console](https://console.aws.amazon.com/)
2. In the search bar at the top, type **"EC2"** and click on it
3. Make sure you're in the correct **Region** (top-right corner)
   - For India: Choose **"Asia Pacific (Mumbai) ap-south-1"**

## 2.2 — Launch Instance

1. Click the orange **"Launch instance"** button

2. **Name and tags:**
   ```
   Name: blousehub-server
   ```

3. **Application and OS Images (AMI):**
   - Click **"Ubuntu"**
   - Select **"Ubuntu Server 24.04 LTS (HVM), SSD Volume Type"**
   - Architecture: **64-bit (x86)**

4. **Instance type:**
   - Select **`t2.small`** (2 GB RAM) — this is the minimum for running Next.js + Django together
   - ⚠️ `t2.micro` (free tier) has only 1 GB RAM and will likely crash during `npm run build`

5. **Key pair (login):**
   - Click **"Create new key pair"**
   - Key pair name: `blousehub-key`
   - Key pair type: **RSA**
   - Private key file format: **`.pem`**
   - Click **"Create key pair"**
   - ⚠️ **The `.pem` file will download automatically. SAVE IT SAFELY. You cannot download it again!**

6. **Network settings:**
   - Click **"Edit"**
   - Check these boxes:
     - ✅ **Allow SSH traffic from** → `Anywhere (0.0.0.0/0)`
     - ✅ **Allow HTTPS traffic from the internet**
     - ✅ **Allow HTTP traffic from the internet**

7. **Configure storage:**
   - Change to **20 GiB** (gp3)
   - The default 8 GB is not enough for node_modules + Python venv + builds

8. Click **"Launch instance"** 🎉

## 2.3 — Get Your Public IP

1. Go to **EC2 → Instances**
2. Click on your `blousehub-server` instance
3. Copy the **Public IPv4 address** (e.g., `3.110.45.123`)
4. We'll refer to this as **`YOUR_EC2_IP`** throughout the guide

## 2.4 — Allocate an Elastic IP (IMPORTANT!)

> Without an Elastic IP, your server's IP changes every time you stop/start the instance.

1. Go to **EC2 → Elastic IPs** (left sidebar under "Network & Security")
2. Click **"Allocate Elastic IP address"**
3. Click **"Allocate"**
4. Select the newly created Elastic IP
5. Click **Actions → "Associate Elastic IP address"**
6. Choose your `blousehub-server` instance
7. Click **"Associate"**
8. **Note the Elastic IP** — this is your new permanent `YOUR_EC2_IP`

---

---

# STEP 3: Connect to Your EC2 Instance via SSH

> ⏱️ Time: 5 minutes

## 3.1 — From Windows (PowerShell)

First, move your `.pem` key to a safe location and fix permissions:

```powershell
# Move the key to your user directory
Move-Item "$env:USERPROFILE\Downloads\blousehub-key.pem" "$env:USERPROFILE\.ssh\blousehub-key.pem"

# Connect to EC2
ssh -i "$env:USERPROFILE\.ssh\blousehub-key.pem" ubuntu@YOUR_EC2_IP
```

> Replace `YOUR_EC2_IP` with your actual Elastic IP address.

## 3.2 — First time connection

When you see:
```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Type **`yes`** and press Enter.

You should now see:
```
ubuntu@ip-172-XX-XX-XX:~$
```

**🎉 You are now inside your EC2 server!**

---

---

# STEP 4: Install All Dependencies on EC2

> ⏱️ Time: 10-15 minutes
>
> Run ALL commands below inside your EC2 terminal (via SSH)

## 4.1 — Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

> This updates all existing packages. Type `Y` if prompted.

## 4.2 — Install Python 3 & pip

```bash
sudo apt install -y python3 python3-pip python3-venv
```

Verify:
```bash
python3 --version
```
Expected output: `Python 3.12.x`

## 4.3 — Install Node.js 20 (LTS)

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:
```bash
node --version
npm --version
```
Expected output: `v20.x.x` and `10.x.x`

## 4.4 — Install Nginx

```bash
sudo apt install -y nginx
```

Verify:
```bash
nginx -v
```

Check Nginx is running:
```bash
sudo systemctl status nginx
```

> At this point, if you visit `http://YOUR_EC2_IP` in your browser, you should see the **"Welcome to Nginx"** page.

## 4.5 — Install PM2 (Process Manager for Next.js)

```bash
sudo npm install -g pm2
```

Verify:
```bash
pm2 --version
```

## 4.6 — Install Git

```bash
sudo apt install -y git
```

---

---

# STEP 5: Set Up the Backend (Django + Gunicorn)

> ⏱️ Time: 15-20 minutes

## 5.1 — Create the project directory

```bash
sudo mkdir -p /var/www
cd /var/www
```

## 5.2 — Clone the backend repo

```bash
sudo git clone https://github.com/prathmesh-lohar/BlouseHub_BackEnd.git backend
```

## 5.3 — Set ownership to your user

```bash
sudo chown -R ubuntu:ubuntu /var/www/backend
```

## 5.4 — Navigate into the project

```bash
cd /var/www/backend
```

## 5.5 — Create a Python virtual environment

```bash
python3 -m venv venv
```

> This creates an isolated Python environment so your project dependencies don't conflict with system Python packages.

## 5.6 — Activate the virtual environment

```bash
source venv/bin/activate
```

> You should see `(venv)` at the beginning of your terminal prompt. This means the venv is active.

## 5.7 — Upgrade pip & install dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn
```

> **Gunicorn** is the production WSGI server that will run Django. Django's built-in `runserver` is NOT suitable for production.

## 5.8 — Create the production `.env` file

```bash
nano /var/www/backend/.env
```

Paste the following (edit the values!):

```env
# Environment Mode
ENVIRONMENT=production

# IMPORTANT: Set to False in production!
DEBUG=False

# Generate a unique secret key (see below)
SECRET_KEY="paste-your-generated-key-here"

# Backend Hostnames
DEV_HOST=127.0.0.1
PROD_HOST=YOUR_EC2_IP

# Allowed Hosts — add your domain if you have one
ALLOWED_HOSTS=YOUR_EC2_IP,your-domain.com,www.your-domain.com,localhost,127.0.0.1

# Razorpay Credentials (use LIVE keys for production)
RAZORPAY_KEY_ID=your_razorpay_key
RAZORPAY_KEY_SECRET=your_razorpay_secret
```

**To generate a secret key**, run this in another terminal or before saving:
```bash
python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

Copy the output and paste it as the `SECRET_KEY` value.

**Save the file:** Press `Ctrl + X`, then `Y`, then `Enter`

## 5.9 — Run database migrations

```bash
python manage.py migrate
```

> This creates all the database tables. Since you're using SQLite, the `db.sqlite3` file will be created automatically.

## 5.10 — Collect static files

```bash
python manage.py collectstatic --noinput
```

> This gathers all static files (CSS, JS, images) into one directory so Nginx can serve them efficiently.

## 5.11 — Create a superuser (admin account)

```bash
python manage.py createsuperuser
```

> Follow the prompts to set a username, email, and password. You'll use this to log into `/admin/`.

## 5.12 — Test that Django works

```bash
gunicorn backend.wsgi:application --bind 0.0.0.0:8000
```

> Visit `http://YOUR_EC2_IP:8000/api/` in your browser. If you see a response, it works!
> Press `Ctrl + C` to stop it.

## 5.13 — Create a systemd service for Gunicorn

This makes Django run automatically in the background and restart on crashes/reboots.

### Create the log directory:
```bash
sudo mkdir -p /var/log/gunicorn
sudo chown ubuntu:ubuntu /var/log/gunicorn
```

### Create the service file:
```bash
sudo nano /etc/systemd/system/blousehub-backend.service
```

Paste the following:

```ini
[Unit]
Description=BlouseHub Django Backend (Gunicorn)
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/var/www/backend
ExecStart=/var/www/backend/venv/bin/gunicorn backend.wsgi:application \
    --bind 127.0.0.1:8000 \
    --workers 3 \
    --timeout 120 \
    --access-logfile /var/log/gunicorn/access.log \
    --error-logfile /var/log/gunicorn/error.log
EnvironmentFile=/var/www/backend/.env
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

**Save:** `Ctrl + X` → `Y` → `Enter`

### Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable blousehub-backend
sudo systemctl start blousehub-backend
```

### Verify it's running:
```bash
sudo systemctl status blousehub-backend
```

Expected output (look for **"active (running)"**):
```
● blousehub-backend.service - BlouseHub Django Backend (Gunicorn)
     Loaded: loaded
     Active: active (running) since ...
```

> ⚠️ If it says "failed", check the logs:
> ```bash
> sudo journalctl -u blousehub-backend -n 50 --no-pager
> ```

---

---

# STEP 6: Set Up the Frontend (Next.js + PM2)

> ⏱️ Time: 10-15 minutes

## 6.1 — Clone the frontend repo

```bash
cd /var/www
sudo git clone https://github.com/prathmesh-lohar/BlouseHub_FrontEnd.git frontend
sudo chown -R ubuntu:ubuntu /var/www/frontend
cd /var/www/frontend
```

## 6.2 — Install Node.js dependencies

```bash
npm ci
```

> `npm ci` is faster and more reliable than `npm install` for production — it installs exact versions from `package-lock.json`.

## 6.3 — Create the production environment file

```bash
nano /var/www/frontend/.env.production
```

Paste the following (edit the values!):

```env
# Environment Mode
NEXT_PUBLIC_APP_ENV=production

# Development URLs (used for local testing — keep as-is)
NEXT_PUBLIC_DEV_BACKEND_HOST=http://127.0.0.1:8000
NEXT_PUBLIC_DEV_API_URL=http://127.0.0.1:8000/api

# Production URLs — CHANGE THESE!
# If you have a domain:
NEXT_PUBLIC_PROD_BACKEND_HOST=https://your-domain.com
NEXT_PUBLIC_PROD_API_URL=https://your-domain.com/api

# If you DON'T have a domain yet (use IP temporarily):
# NEXT_PUBLIC_PROD_BACKEND_HOST=http://YOUR_EC2_IP
# NEXT_PUBLIC_PROD_API_URL=http://YOUR_EC2_IP/api

# Razorpay Credentials
NEXT_PUBLIC_RAZORPAY_KEY_ID="your_razorpay_key"
NEXT_PUBLIC_RAZORPAY_KEY_SECRET="your_razorpay_secret"
```

**Save:** `Ctrl + X` → `Y` → `Enter`

## 6.4 — Build the Next.js app

```bash
npm run build
```

> ⏱️ This may take 1-3 minutes. It compiles your Next.js app into optimized production files.
>
> ⚠️ If the build fails with "out of memory", your instance needs more RAM. Either:
> - Upgrade to `t2.small` or larger
> - OR create a swap file (temporary fix):
>   ```bash
>   sudo fallocate -l 2G /swapfile
>   sudo chmod 600 /swapfile
>   sudo mkswap /swapfile
>   sudo swapon /swapfile
>   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
>   ```
>   Then retry `npm run build`.

## 6.5 — Start Next.js with PM2

```bash
pm2 start npm --name "blousehub-frontend" -- start
```

Verify it's running:
```bash
pm2 status
```

You should see:
```
┌─────┬──────────────────────┬─────────────┬──────┬───────────┐
│ id  │ name                 │ status      │ cpu  │ memory    │
├─────┼──────────────────────┼─────────────┼──────┼───────────┤
│ 0   │ blousehub-frontend   │ online      │ 0%   │ 80MB      │
└─────┴──────────────────────┴─────────────┴──────┴───────────┘
```

## 6.6 — Save PM2 process list & enable auto-start on reboot

```bash
pm2 save
pm2 startup
```

> PM2 will print a command like:
> ```
> sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu
> ```
> **Copy and paste that exact command and run it.** This ensures Next.js starts automatically after a server reboot.

## 6.7 — Test that Next.js works

```bash
curl http://localhost:3000
```

> If you see HTML output, Next.js is running correctly!

---

---

# STEP 7: Configure Nginx (Reverse Proxy)

> ⏱️ Time: 10 minutes
>
> Nginx will sit in front of both apps and route traffic to the correct one.

## 7.1 — Create the Nginx configuration file

```bash
sudo nano /etc/nginx/sites-available/blousehub
```

Paste the following (replace `YOUR_EC2_IP` and domain):

```nginx
server {
    listen 80;
    server_name YOUR_EC2_IP your-domain.com www.your-domain.com;
    # ↑ Replace with your actual EC2 IP and domain (if you have one)
    # If no domain, just use: server_name YOUR_EC2_IP;

    # ========================
    # FRONTEND — Next.js
    # ========================
    # All page requests go to Next.js
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # ========================
    # BACKEND — Django API
    # ========================
    # All API requests go to Django/Gunicorn
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ========================
    # BACKEND — Django Admin Panel
    # ========================
    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # ========================
    # STATIC FILES — Served directly by Nginx (fast!)
    # ========================
    location /static/ {
        alias /var/www/backend/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # ========================
    # MEDIA FILES — User uploads
    # ========================
    location /media/ {
        alias /var/www/backend/media/;
        expires 7d;
        add_header Cache-Control "public";
    }

    # ========================
    # SECURITY HEADERS
    # ========================
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Max upload size (for images, etc.)
    client_max_body_size 10M;
}
```

**Save:** `Ctrl + X` → `Y` → `Enter`

## 7.2 — Enable the site

```bash
# Create a symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/blousehub /etc/nginx/sites-enabled/

# Remove the default Nginx page
sudo rm /etc/nginx/sites-enabled/default
```

## 7.3 — Test the Nginx configuration

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> ⚠️ If you see errors, double-check the config file for typos (especially missing semicolons).

## 7.4 — Restart Nginx

```bash
sudo systemctl restart nginx
```

## 7.5 — TEST EVERYTHING! 🎉

Open your browser and try:

| URL | What you should see |
|-----|---------------------|
| `http://YOUR_EC2_IP` | Your Next.js frontend (BlouseHub homepage) |
| `http://YOUR_EC2_IP/api/` | Django REST API response |
| `http://YOUR_EC2_IP/admin/` | Django admin login page |

> ⚠️ **If you get a 502 Bad Gateway error:**
> - Check backend: `sudo systemctl status blousehub-backend`
> - Check frontend: `pm2 status`
> - Check Nginx logs: `sudo tail -20 /var/log/nginx/error.log`

---

---

# STEP 8: Set Up HTTPS with Let's Encrypt (Free SSL)

> ⏱️ Time: 5 minutes
>
> ⚠️ **REQUIRES A DOMAIN NAME** — You cannot get SSL for a bare IP address.
> If you don't have a domain, skip this step (your site will work on HTTP).

## 8.1 — Point your domain to EC2

Go to your domain registrar (GoDaddy, Namecheap, Hostinger, etc.) and add:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | @ | YOUR_EC2_IP | 300 |
| A | www | YOUR_EC2_IP | 300 |

> Wait 5-10 minutes for DNS to propagate. Test with:
> ```bash
> ping your-domain.com
> ```

## 8.2 — Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

## 8.3 — Get SSL certificate

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Follow the prompts:
1. Enter your email address
2. Agree to terms of service → `Y`
3. Share email with EFF → `N` (optional)

> Certbot will automatically modify your Nginx config to add SSL settings.

## 8.4 — Verify auto-renewal

```bash
sudo certbot renew --dry-run
```

> Certbot sets up a cron job to renew your certificate automatically every 90 days.

## 8.5 — Test HTTPS

Open `https://your-domain.com` in your browser. You should see a 🔒 lock icon!

---

---

# STEP 9: Set Up GitHub Actions CI/CD (SSH Deployment)

> ⏱️ Time: 15-20 minutes
>
> **Simple approach:** GitHub Actions SSHs into your EC2 and runs deploy commands directly. No extra AWS services needed!

---

## STEP 9A: Set Up SSH Access for GitHub Actions

> ⏱️ Time: 5 minutes

### 9A.1 — Prepare your SSH key

You already have the `blousehub-key.pem` file from Step 2. GitHub Actions will use this same key to SSH into your EC2.

1. Open the `.pem` file and **copy its entire contents**:

**Windows (PowerShell):**
```powershell
Get-Content "$env:USERPROFILE\.ssh\blousehub-key.pem" | Set-Clipboard
```

> The key content looks like:
> ```
> -----BEGIN RSA PRIVATE KEY-----
> MIIEpAIBAAKCAQEA...
> (many lines of random characters)
> ...
> -----END RSA PRIVATE KEY-----
> ```

### 9A.2 — Ensure Git is set up on EC2

**SSH into your EC2** and configure Git so your server can pull from GitHub:

```bash
# Option A: For PUBLIC repos (no authentication needed)
cd /var/www/backend
git remote set-url origin https://github.com/prathmesh-lohar/BlouseHub_BackEnd.git

cd /var/www/frontend
git remote set-url origin https://github.com/prathmesh-lohar/BlouseHub_FrontEnd.git
```

```bash
# Option B: For PRIVATE repos (use a GitHub Personal Access Token)
# 1. Go to GitHub → Settings → Developer settings → Personal Access Tokens → Generate new token
# 2. Give it repo access
# 3. Use this URL format:
cd /var/www/backend
git remote set-url origin https://YOUR_GITHUB_TOKEN@github.com/prathmesh-lohar/BlouseHub_BackEnd.git

cd /var/www/frontend
git remote set-url origin https://YOUR_GITHUB_TOKEN@github.com/prathmesh-lohar/BlouseHub_FrontEnd.git
```

---

## STEP 9B: Add GitHub Secrets to BOTH Repos

> ⏱️ Time: 5 minutes

### For the Backend repo (`BlouseHub_BackEnd`):

1. Go to [https://github.com/prathmesh-lohar/BlouseHub_BackEnd](https://github.com/prathmesh-lohar/BlouseHub_BackEnd)
2. Click **Settings → Secrets and variables → Actions**
3. Click **"New repository secret"** and add these one by one:

| Secret Name | Value |
|-------------|-------|
| `EC2_HOST` | Your Elastic IP address (e.g., `3.110.45.123`) |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | The **entire contents** of your `blousehub-key.pem` file (including the `-----BEGIN...` and `-----END...` lines) |

### For the Frontend repo (`BlouseHub_FrontEnd`):

1. Go to [https://github.com/prathmesh-lohar/BlouseHub_FrontEnd](https://github.com/prathmesh-lohar/BlouseHub_FrontEnd)
2. Add the **same 3 secrets** as above

> ⚠️ **Common mistake:** When pasting the SSH key, make sure there are **no extra spaces or blank lines** at the end. Copy it exactly as-is.

---

## STEP 9C: Create Backend Deployment Workflow

> ⏱️ Time: 5 minutes

### 📁 Your backend folder structure after this step:

```
BlouseHub_BackEnd/
├── .github/
│   └── workflows/
│       └── deploy.yml          ← GitHub Actions workflow (SSH deploy)
├── backend/
├── apps/
├── manage.py
├── requirements.txt
└── ...
```

> 💡 **No `appspec.yml` or `scripts/` folder needed!** Everything runs directly via SSH.

---

### File: `.github/workflows/deploy.yml`

> This workflow SSHs into your EC2 and runs all deployment commands.

Create: `c:\BlouseHub\backend\.github\workflows\deploy.yml`

```yaml
name: Deploy Backend to EC2

on:
  push:
    branches: [master]

jobs:
  deploy:
    name: 🚀 Deploy Django Backend via SSH
    runs-on: ubuntu-latest

    steps:
      - name: 📦 Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v1.2.2
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            echo "🔄 Starting backend deployment..."

            # Navigate to project
            cd /var/www/backend

            # Pull latest code from GitHub
            git pull origin master

            # Activate virtual environment
            source venv/bin/activate

            # Upgrade pip
            pip install --upgrade pip

            # Install any new dependencies
            pip install -r requirements.txt

            # Run database migrations
            python manage.py migrate --noinput

            # Collect static files
            python manage.py collectstatic --noinput

            # Restart Gunicorn service
            sudo systemctl restart blousehub-backend

            # Wait for service to start
            sleep 3

            # Verify it's running
            if sudo systemctl is-active --quiet blousehub-backend; then
                echo "✅ Backend deployment complete!"
            else
                echo "❌ Backend failed to start!"
                sudo journalctl -u blousehub-backend -n 20 --no-pager
                exit 1
            fi
```

### Allow passwordless sudo (needed for restarting services)

**SSH into EC2** and run:

```bash
sudo visudo
```

Add this line at the bottom:

```
ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl daemon-reload, /bin/systemctl restart blousehub-backend, /bin/systemctl status blousehub-backend, /bin/systemctl is-active blousehub-backend
```

**Save:** `Ctrl + X` → `Y` → `Enter`

---

## STEP 9D: Create Frontend Deployment Workflow

> ⏱️ Time: 5 minutes

### 📁 Your frontend folder structure after this step:

```
BlouseHub_FrontEnd/
├── .github/
│   └── workflows/
│       └── deploy.yml          ← GitHub Actions workflow (SSH deploy)
├── src/
├── package.json
└── ...
```

---

### File: `.github/workflows/deploy.yml`

Create: `c:\BlouseHub\frontend\.github\workflows\deploy.yml`

```yaml
name: Deploy Frontend to EC2

on:
  push:
    branches: [master]

jobs:
  deploy:
    name: 🚀 Deploy Next.js Frontend via SSH
    runs-on: ubuntu-latest

    steps:
      - name: 📦 Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v1.2.2
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script_stop: true
          command_timeout: 10m
          script: |
            echo "🔄 Starting frontend deployment..."

            # Navigate to project
            cd /var/www/frontend

            # Pull latest code from GitHub
            git pull origin master

            # Install Node.js dependencies
            echo "📦 Installing dependencies..."
            npm ci

            # Build the Next.js app
            echo "🏗️ Building Next.js application..."
            npm run build

            # Restart PM2 process
            if pm2 describe blousehub-frontend > /dev/null 2>&1; then
                echo "♻️ Restarting existing PM2 process..."
                pm2 restart blousehub-frontend
            else
                echo "🆕 Starting new PM2 process..."
                pm2 start npm --name "blousehub-frontend" -- start
            fi

            # Save PM2 process list
            pm2 save

            # Wait for app to start
            sleep 5

            # Verify
            if pm2 describe blousehub-frontend | grep -q "online"; then
                echo "✅ Frontend deployment complete!"
            else
                echo "❌ Frontend failed to start!"
                pm2 logs blousehub-frontend --lines 20 --nostream
                exit 1
            fi
```

> ⚠️ **Note:** `command_timeout: 10m` is set because `npm run build` can take several minutes. Without this, the SSH connection may timeout.

### Allow passwordless sudo for PM2 commands (if needed)

PM2 runs as the `ubuntu` user, so most PM2 commands don't need sudo. However, if you add Nginx restart commands to the workflow, add:

```bash
sudo visudo
```

Add:
```
ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl reload nginx, /bin/systemctl restart nginx
```

---

## STEP 9E: Push the Workflow Files

### ⚠️ Commit and push to trigger the first deployment:

**Backend:**
```powershell
cd c:\BlouseHub\backend
git add .github/workflows/deploy.yml
git commit -m "Add SSH-based CI/CD deployment workflow"
git push origin master
```

**Frontend:**
```powershell
cd c:\BlouseHub\frontend
git add .github/workflows/deploy.yml
git commit -m "Add SSH-based CI/CD deployment workflow"
git push origin master
```

---

---

# STEP 10: Test & Monitor Deployments

## 10.1 — Watch GitHub Actions

1. Go to your GitHub repo → **Actions** tab
2. You should see the workflow running
3. Click on it to see live logs
4. It should:
   - ✅ SSH into EC2
   - ✅ Pull latest code
   - ✅ Install dependencies
   - ✅ Build (frontend) / Migrate (backend)
   - ✅ Restart services
   - ✅ Verify the app is running

## 10.2 — Verify your site

| Test | URL | Expected Result |
|------|-----|-----------------|
| Frontend | `http://YOUR_EC2_IP` | BlouseHub homepage loads |
| API | `http://YOUR_EC2_IP/api/` | Django REST API responds |
| Admin | `http://YOUR_EC2_IP/admin/` | Django admin login page |

> ⚠️ **If the GitHub Actions step fails:**
>
> **Check the GitHub Actions log** for error messages. Common issues:
>
> | Error | Fix |
> |-------|-----|
> | `ssh: connect to host ... port 22: Connection timed out` | Check EC2 Security Group allows SSH (port 22) from `0.0.0.0/0` |
> | `Permission denied (publickey)` | Verify the `EC2_SSH_KEY` secret contains the complete `.pem` file contents |
> | `Host key verification failed` | The `appleboy/ssh-action` handles this automatically — make sure you're on `v1.2.2` or later |
> | `sudo: a password is required` | Run the `visudo` step from 9C to allow passwordless sudo |
> | `npm run build` timeout | Increase `command_timeout` or add swap space (see Troubleshooting) |

## 10.3 — Test the full pipeline

1. Make a small change in your code
2. Push to `master`:
   ```bash
   git add .
   git commit -m "Test SSH deployment pipeline"
   git push origin master
   ```
3. Watch **GitHub Actions** run ✅
4. Refresh your site — the change should be live! 🎉

---

---

# 📋 Quick Reference — Useful Commands

```bash
# ============================
# BACKEND (Django/Gunicorn)
# ============================
sudo systemctl status blousehub-backend      # Check if running
sudo systemctl restart blousehub-backend     # Restart
sudo systemctl stop blousehub-backend        # Stop
sudo systemctl start blousehub-backend       # Start
sudo journalctl -u blousehub-backend -f      # View live logs
tail -f /var/log/gunicorn/error.log          # Gunicorn error logs

# Activate Django venv manually
cd /var/www/backend && source venv/bin/activate

# ============================
# FRONTEND (Next.js/PM2)
# ============================
pm2 status                          # Check all PM2 processes
pm2 restart blousehub-frontend     # Restart
pm2 stop blousehub-frontend        # Stop
pm2 logs blousehub-frontend        # View live logs

# ============================
# NGINX
# ============================
sudo nginx -t                       # Test config for errors
sudo systemctl restart nginx        # Restart
sudo systemctl reload nginx         # Reload without downtime
sudo tail -f /var/log/nginx/error.log     # Error logs
sudo tail -f /var/log/nginx/access.log    # Access logs

# ============================
# SYSTEM
# ============================
df -h               # Check disk space
free -h             # Check memory usage
htop                # Interactive process viewer (install: sudo apt install htop)
```

---

---

# 💰 Estimated Monthly Cost

| Resource | Cost (USD) |
|----------|-----------:|
| EC2 `t2.small` (on-demand, Mumbai) | ~$16.50/mo |
| Elastic IP (attached to running instance) | Free |
| Storage (20 GB gp3) | ~$1.60/mo |
| Data Transfer (first 100 GB/mo) | Free |
| Let's Encrypt SSL | Free |
| GitHub Actions (2,000 free mins/mo) | Free |
| **Total** | **~$18/mo** |

> 💡 **Save money:**
> - Buy a 1-year Reserved Instance for `t3.small` → drops to ~$9/mo

---

---

# 🔧 Troubleshooting

## Problem: 502 Bad Gateway

**Cause:** Nginx can't reach the backend or frontend app.

```bash
# Check if Django is running
sudo systemctl status blousehub-backend

# Check if Next.js is running
pm2 status

# If either is down, restart:
sudo systemctl restart blousehub-backend
pm2 restart blousehub-frontend
```

## Problem: `npm run build` kills the server (out of memory)

**Fix:** Add swap space:
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Problem: GitHub Actions fails with "Permission denied"

**Fix:** Make sure:
1. The `.pem` key is correctly pasted into the `EC2_SSH_KEY` GitHub secret
2. The key doesn't have extra spaces or line breaks
3. The EC2 Security Group allows SSH (port 22) from `0.0.0.0/0`

## Problem: Site not accessible after EC2 restart

**Fix:** Check that services are running:
```bash
sudo systemctl start blousehub-backend
pm2 resurrect
sudo systemctl start nginx
```

## Problem: Django static files not loading (404 on /static/)

**Fix:** Run collectstatic and check the Nginx alias path:
```bash
cd /var/www/backend && source venv/bin/activate
python manage.py collectstatic --noinput
ls -la /var/www/backend/staticfiles/    # This directory must exist
sudo systemctl reload nginx
```

## Problem: `git pull` fails on EC2

**Fix:** If you see "Permission denied" or authentication errors:
```bash
# For public repos — make sure the URL is HTTPS (not SSH):
git remote set-url origin https://github.com/prathmesh-lohar/BlouseHub_BackEnd.git

# For private repos — use a Personal Access Token:
git remote set-url origin https://YOUR_TOKEN@github.com/prathmesh-lohar/BlouseHub_BackEnd.git
```

---

**🎉 Congratulations! Your BlouseHub app is now live on AWS with automated SSH deployments!**
