# MERN App on AWS — Deployment Blueprint (Phase 1)

This document captures **what we have done so far** and the **remaining steps** to fully deploy the MERN CRUD app on AWS, strictly following the reference architecture diagram.

---

## ✅ What We Have Done (Phase 1 — Foundation)

### 1. VPC + Networking
- Created a **VPC** with CIDR block `10.0.0.0/16` (example).
- Added **subnets** (public subnet `10.0.1.0/24` for EC2).
- Configured a **Route Table** with:
  - Local routes for VPC CIDR.
  - `0.0.0.0/0` → **Internet Gateway** (IGW).
- Attached **Internet Gateway** (IGW) to the VPC.

### 2. Security Groups
- Created **Security Group** for EC2 with inbound rules:
  - **SSH (22)** → only from your IP (admin access).
  - **HTTP (80)** → 0.0.0.0/0 (web traffic).
  - **HTTPS (443)** → 0.0.0.0/0 (secure web traffic).

### 3. Elastic IP + EC2
- Allocated and associated an **Elastic IP** to the EC2 instance (IP: `23.21.22.150`).
- Launched **EC2 instance (Ubuntu)** inside the VPC public subnet.
- Verified SSH access with `.pem` key.

### 4. DNS (Route 53 + Registrar)
- Created **Route 53 Hosted Zone** for `lauv.in`.
- Updated **nameservers** in GoDaddy to Route 53’s 4 NS records.
- Added **A Record** in Route 53:
  - `api.lauv.in` → Elastic IP (`23.21.22.150`).

✅ At this stage:
- `api.lauv.in` resolves to your EC2 server.

---

## 🚧 What’s Left (Strictly Following the Diagram)

### Phase 1 (MVP – Basic Live Setup)
1. **Prep EC2**
   - Update Ubuntu (`apt update && apt upgrade`).
   - Install **Nginx, Node.js, PM2, Git, UFW**.
   - Enable UFW firewall with SSH + HTTP + HTTPS.

2. **Deploy Backend (API)**
   - Copy or clone backend code into `/opt/mern-api`.
   - Install dependencies (`npm ci`).
   - Configure `.env` with:
     - `PORT=5000`
     - `MONGO_URI=<MongoDB Atlas connection string>`
   - Start app with **PM2** and enable autostart.
   - Test locally on EC2: `curl http://127.0.0.1:5000/api/health`.

3. **Configure Nginx (Reverse Proxy for API)**
   - Create server block for `api.lauv.in`:
     - HTTP → HTTPS redirect.
     - Proxy `/api/*` to `http://127.0.0.1:5000`.

4. **Install Let’s Encrypt (Certbot)**
   - Issue TLS cert for `api.lauv.in`.
   - Enable auto-renew cronjob.

---

### Phase 2 (Frontend via CloudFront + S3)
1. **S3 Bucket**
   - Create private bucket `app.lauv.in`.
   - Enable versioning (optional).

2. **ACM Certificate**
   - Request cert for `app.lauv.in` in **us-east-1**.
   - DNS validation via Route 53.

3. **CloudFront**
   - Create distribution with:
     - Origin: S3 bucket (private, OAC enabled).
     - Alternate domain: `app.lauv.in`.
     - Viewer policy: Redirect HTTP → HTTPS.
     - TLS: attach ACM cert.
   - Route 53 record:
     - `app.lauv.in` → CloudFront (Alias A record).

4. **Build & Deploy React App**
   - `npm run build` → Upload `build/` to S3 (`aws s3 sync`).
   - Invalidate CloudFront cache (`aws cloudfront create-invalidation`).

---

### Phase 3 (Ops + Observability)
1. **CloudWatch Alarms**
   - CPU utilization > 80% for 5 minutes → alert.
   - Status check failed → alert.

2. **Snapshots**
   - Daily AMI snapshot of EC2 for backup.

3. **Atlas Monitoring**
   - Metrics for query latency, connections.

---

## 🔄 Current Network Flow

```
Browser → api.lauv.in (DNS → Elastic IP → EC2 → Nginx → Node.js → MongoDB Atlas)
```

---

## 🎯 End of Phase 1 Goal

By finishing Phase 1, you will have:
- `https://api.lauv.in/api/health` working with Let’s Encrypt TLS.
- Backend CRUD connected to MongoDB Atlas.
- Ready to extend to Phase 2 → S3 + CloudFront for React frontend.




# MERN on AWS — Stage 2 Runbook (Backend HTTPS + Frontend on CloudFront)

> **Goal:** Make `api.lauv.in` (backend) live on HTTPS via Nginx + Let's Encrypt, and prepare to serve the React frontend via S3 + CloudFront exactly as in the architecture diagram.

---

## Prerequisites (checked)
- Route 53 hosted zone for your domain (e.g., `lauv.in`) is active.
- `api.lauv.in` A record → **Elastic IP** of your EC2.
- MongoDB Atlas has **allowlisted** your EC2 **Elastic IP**.
- EC2 (Ubuntu 22.04) is reachable via SSH.

> If you use Amazon Linux 2023, the reasoning is identical; package commands differ slightly.
 
---

## Part A — Backend on EC2 with HTTPS

### A1) Update OS and install base tooling
**Why:** security patches + baseline web stack.

```bash
ssh -i /path/to/key.pem ubuntu@<EIP>

sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git ufw
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm i -g pm2
node -v && npm -v && pm2 -v
```

**UFW firewall (defense in depth)**  
Even with Security Groups, host firewall is a second guardrail.
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'   # opens 80/443
sudo ufw --force enable
```

---

### A2) Layout code on disk and keep secrets out of Git
**Why:** predictable paths + secure configuration.

```bash
sudo mkdir -p /opt/mern-api
sudo chown -R ubuntu:ubuntu /opt/mern-api
cd /opt/mern-api

# Option: clone your repo
git clone <YOUR_REPO_URL> .

cd backend
npm ci

# Configure runtime secrets locally on the server (never commit .env)
cat > .env <<'EOF'
PORT=5000
MONGO_URI=mongodb+srv://<user>:<pass>@<cluster>/<db>?retryWrites=true&w=majority
EOF
```

---

### A3) Start the API with PM2 and persist across reboots
**Why:** PM2 auto-restarts on crash and on reboot.

```bash
pm2 start server.js --name mern-api --env production
pm2 save
pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

**Validate locally on the instance:**
```bash
curl -s http://127.0.0.1:5000/api/health
# expect: {"ok":true,...}
```

**Troubleshooting tips:**
```bash
pm2 logs mern-api --lines 100
sudo lsof -i :5000      # confirm Node is listening
```

---

### A4) Configure Nginx as reverse proxy for `api.lauv.in`
**Why:** Terminate TLS at Nginx, keep Node private on loopback, add headers/timeouts.

Create the server block:
```bash
sudo tee /etc/nginx/sites-available/api.conf >/dev/null <<'NGINX'
# Force HTTPS
server {
  listen 80;
  server_name api.lauv.in;
  location / { return 301 https://$host$request_uri; }
}

# HTTPS reverse proxy
server {
  listen 443 ssl http2;
  server_name api.lauv.in;

  # Certbot will inject ssl_certificate directives here.

  # Basic hardening headers
  add_header X-Content-Type-Options nosniff;
  add_header X-Frame-Options DENY;
  add_header X-XSS-Protection "1; mode=block";

  # Proxy all API requests to Node
  location /api/ {
    proxy_pass http://127.0.0.1:5000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_connect_timeout 5s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
  }
}
NGINX

sudo ln -s /etc/nginx/sites-available/api.conf /etc/nginx/sites-enabled/ || true
sudo nginx -t && sudo systemctl reload nginx
```

**Why the split 80/443?**  
- Port 80 is kept only to redirect to HTTPS and allow HTTP-01 challenges if needed.  
- Port 443 serves encrypted traffic and proxies to the local Node process.

---

### A5) Issue a free TLS certificate (Let’s Encrypt)
**Why:** HTTPS is required for security, browser trust, and modern APIs.

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.lauv.in
# Follow prompts: email, agree, force redirect to HTTPS.
# Certs are stored under /etc/letsencrypt/live/api.lauv.in/
```

**Auto-renew nightly:**
```bash
echo "0 3 * * * root certbot renew --quiet" | sudo tee /etc/cron.d/certbot_renew
```

**Validate from your laptop:**
```bash
curl -I https://api.lauv.in/api/health
# 200 OK with a valid certificate chain
```

> **Why certbot and not ACM?** ALB/CloudFront can use ACM directly, but for an EC2 + Nginx public endpoint, Let’s Encrypt via certbot is the simplest path.

---

### A6) Tighten CORS (optional but recommended)
**Why:** only allow requests from your frontend origin.
```js
// server.js
import cors from "cors";
app.use(cors({ origin: "https://app.lauv.in" }));
```
```bash
pm2 restart mern-api
```

**At this point:**  
`https://api.lauv.in/api/health` is live over HTTPS and the backend talks to MongoDB Atlas using your `MONGO_URI`.

---

## Part B — Frontend via S3 + CloudFront (as per diagram)

### B1) Build React with the correct API base
**Why:** the compiled bundle must know the real API URL.
```bash
cd frontend
export REACT_APP_API_BASE=https://api.lauv.in
npm ci
npm run build
```

---

### B2) Create a private S3 bucket for the build
**Why:** keep objects private; only CloudFront (OAC) may read.
- Bucket name: `app.lauv.in` (same region as you prefer; naming is global).
- Block Public Access: **ON**.
- Versioning: optional but recommended.

---

### B3) Request an ACM certificate in **us-east-1**
**Why:** CloudFront requires certs from the N. Virginia region.
- ACM (us-east-1) → Request certificate → `app.lauv.in` → DNS validation.
- Route 53 will add the CNAME automatically.
- Wait until status is **Issued**.

---

### B4) Create a CloudFront distribution with OAC
**Why:** CDN for speed; secure origin access to S3.

- Origin: the `app.lauv.in` S3 bucket.  
- **Origin Access Control (OAC)**: create & attach.  
- Default root object: `index.html`.  
- Viewer protocol policy: **Redirect HTTP to HTTPS**.  
- Alternate domain name (CNAME): `app.lauv.in`.  
- Custom SSL cert: select the ACM cert from B3.  
- (SPA) Optional custom error responses: map 403/404 → `/index.html` (HTTP 200).

**Update the S3 bucket policy** when prompted so only this OAC principal can read.

---

### B5) Route 53 record for the app
**Why:** friendly name for CloudFront.
- Create **A (ALIAS)** record: `app.lauv.in` → your CloudFront distribution.

---

### B6) Upload the build & invalidate cache
**Why:** publish the UI and flush stale CDN copies.

```bash
aws s3 sync frontend/build/ s3://app.lauv.in/ --delete

# After each deploy:
aws cloudfront create-invalidation \
  --distribution-id <DIST_ID> \
  --paths "/*"
```

**Validate:**
```bash
curl -I https://app.lauv.in     # should include: Via: cloudfront
# Open app in browser, check Network calls go to https://api.lauv.in
```

---

## Part C — Minimal Ops (matching the diagram)

### C1) CloudWatch alarms (quick wins)
**Why:** get notified if the instance is sick.
- **CPUUtilization > 80% for 5m** → email/SNS.
- **StatusCheckFailed > 0 for 1m** → email/SNS.

### C2) Daily AMI snapshot (manual or DLM)
**Why:** rollback safety.
- Create an AMI from the EC2 instance or use **Data Lifecycle Manager** to automate daily AMIs.

### C3) Optional uptime check
**Why:** detect HTTP failures before users do.
- External monitor (e.g., Route 53 Health Check or any third-party) hitting `https://api.lauv.in/api/health` every minute.

---

## Rollback Playbook

- **Frontend:** S3 versioning → restore previous object versions, then invalidate CloudFront.  
- **Backend:** `pm2 rollback` (if using PM2 deploy) or `git revert && pm2 restart mern-api`.  
- **DNS:** Repoint `api.lauv.in` back to previous instance (if blue/green).

---

## Common Pitfalls & Fixes

- **Frontend still calls localhost** → Ensure `REACT_APP_API_BASE` was set *before* `npm run build`.
- **CORS errors** → Set `cors({ origin: "https://app.lauv.in" })` and restart API.
- **502/504 from Nginx** → API not running, port mismatch, or health route path incorrect.
- **Atlas connection failures** → EC2 Elastic IP not in Atlas allowlist, wrong credentials in `MONGO_URI`.
- **CloudFront serving old UI** → missing invalidation after deploy.

---

## Final Network Flow (matches the diagram)

```
Users
 ├─► https://app.lauv.in  → CloudFront (TLS) → S3 (React build via OAC)
 └─► https://api.lauv.in  → EC2 (Nginx TLS) → Node/Express → MongoDB Atlas (TLS)
```

> Next natural upgrade (Phase 2+): move backend to **private subnets** behind an **ALB** (no public IP), add a **NAT** or **VPC endpoints**, and consider **WAF** in front of CloudFront/ALB.


