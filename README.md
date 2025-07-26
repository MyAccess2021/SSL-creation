## Free SSL Certification 


## ✅ Step-by-Step Process

### 1. 🛠️ Build Docker Image for React App

Create a **multi-stage Dockerfile** for a production-ready React app:

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

> 🔧 Ensure `nginx.conf` is properly configured for HTTPS (see below).

---

### 2. 🌐 DNS Setup

Configure the DNS `A` record for your domain to point to your EC2 instance:

```
Type: A
Host: cap.myaccessio.com
Value: <EC2 Public IP> (e.g., 3.110.168.58)
TTL: 300
```

Wait for propagation (can check via `nslookup cap.myaccessio.com`).

---

### 3. 🔐 Get Free SSL Certificate from Let's Encrypt

#### 🧼 Cleanly prepare for Certbot:

1. **Temporarily remove any Nginx site configs** (if any running on host machine):

```bash
sudo rm /etc/nginx/sites-enabled/cap.myaccessio.com
```

2. **Stop Nginx or Docker container using port 80**:

```bash
sudo systemctl stop nginx
# OR
docker stop <container-name>
```

#### 🚀 Run Certbot:

```bash
sudo apt update
sudo apt install certbot
sudo certbot certonly --standalone -d cap.myaccessio.com
```

> If successful, you'll see:
>
> ```
> Congratulations! Your certificate and chain have been saved at:
> /etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem
> ```

3. **Re-enable your Nginx config (if applicable):**

```bash
sudo ln -s /etc/nginx/sites-available/cap.myaccessio.com /etc/nginx/sites-enabled/
```

4. **Test and restart Nginx** (if using systemd on host):

```bash
sudo nginx -t && sudo systemctl start nginx
```

---

### 4. 🧾 Configure Nginx for HTTPS

Place this in your Docker container’s `nginx.conf`:

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

### 5. 🐳 Run Docker Container with HTTPS

Stop old container:

```bash
docker rm -f react-container2
```

Run new container with certificates mounted:

```bash
docker run -d \
  --name react-container2 \
  -p 80:80 -p 443:443 \
  -v /etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem:/etc/letsencrypt/live/cap.myaccessio.com/fullchain.pem:ro \
  -v /etc/letsencrypt/live/cap.myaccessio.com/privkey.pem:/etc/letsencrypt/live/cap.myaccessio.com/privkey.pem:ro \
  myaccess2021/cap:latest
```

---

### 6. 🔁 Auto-Renewal of SSL Certificates

Certbot automatically installs a cron job for renewal. To test:

```bash
sudo certbot renew --dry-run
```

Certificates will be updated without requiring a manual re-issue.

---

## ✅ Final Result

* React app deployed successfully via Docker.
* Available on:

  * `http://cap.myaccessio.com` → Redirects to HTTPS
  * `https://cap.myaccessio.com` → Secure, valid SSL certificate
* Nginx inside Docker handles all HTTP/S traffic.

---

## 🔄 Future Tasks

* 🔁 Rebuild Docker image after React code changes.
* 🚢 Re-deploy updated Docker image.
* 🔒 SSL auto-renewed by Certbot—just ensure port 80 remains available during renewal.

---

