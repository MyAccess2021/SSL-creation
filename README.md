# SSL-creation
---

# 📘 React App Deployment with HTTPS on AWS using Docker and Let's Encrypt

## 🧱 Stack

* **React** (Vite build)
* **Docker**
* **Nginx** (inside Docker)
* **Certbot / Let’s Encrypt** (for HTTPS)
* **Domain**: `cap.myaccessio.com`
* **Hosting**: AWS EC2 (Ubuntu)

---

## ✅ Step-by-Step Process

### 1. 🛠️ Build Docker Image for React App

Created a multi-stage Dockerfile for a production-ready React app:

```Dockerfile
# Stage 1: Build Vite React App
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --legacy-peer-deps
COPY . .
ENV NODE_ENV=production
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

> 🔧 `nginx.conf` configured to serve app and enable HTTPS.

---

### 2. 🔐 Get Free SSL Certificate from Let's Encrypt

Used **Certbot** to get a certificate:

```bash
sudo apt update
sudo apt install certbot
sudo systemctl stop nginx  # Or stop Docker container using port 80
sudo certbot certonly --standalone -d cap.myaccessio.com
```

Cert files saved to:

* `/etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem`
* `/etc/letsencrypt/live/cap.myaccessio.com/privkey.pem`

---

### 3. 🧾 Configure Nginx for HTTPS

Used this inside the container's `nginx.conf`:

```nginx
server {
    listen 80;
    server_name cap.myaccessio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name cap.myaccessio.com;

    ssl_certificate /etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cap.myaccessio.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /usr/share/nginx/html;
    index index.html;

    location /api/ {
        proxy_pass http://email-backend-container:5000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

### 4. 🚀 Run Docker Container with HTTPS

Stopped old container:

```bash
docker rm -f react-container2
```

Ran new container with certificate files mounted:

```bash
docker run -d \
  --name react-container2 \
  -p 80:80 -p 443:443 \
  -v /etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem:/etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem:ro \
  -v /etc/letsencrypt/live/cap.myaccessio.com/privkey.pem:/etc/letsencrypt/live/cap.myaccessio.com/privkey.pem:ro \
  myaccess2021/cap:latest
```

---

### 5. 🌐 DNS Setup

Configured DNS `A` record:

```
Type: A
Host: cap.myaccessio.com
Value: <EC2 Public IP> (e.g., 3.110.168.58)
TTL: 300
```

---

### 6. 🔁 Auto-Renewal (Handled by Certbot)

Certbot created a cron job to auto-renew the certificate.

To test renewal:

```bash
sudo certbot renew --dry-run
```

---

## ✅ Final Result

* React app deployed successfully.
* Available on:

  * `http://cap.myaccessio.com` → Redirects to HTTPS
  * `https://cap.myaccessio.com` → Secure HTTPS with Let's Encrypt
* Nginx inside Docker handles HTTP(S) requests.

---

## 🔄 Future Tasks

* Rebuild Docker image when React code changes.
* Re-deploy with updated image.
* Certificates auto-renewed by Certbot.

---

