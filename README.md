# TravelMemory Deployment Guide


## Deployment Steps

### 1. Launch EC2 Instances

- **Frontend EC2**: Hosts React build and Nginx
- **Backend EC2**: Hosts Node.js app and Nginx reverse proxy

**Open these ports:**
- 22 (SSH)
- 80, 443 (Nginx)
- 3000 (backend internal only)

---

```
    +-------------+
User--| Cloudflare  |---> Load Balancer
    +-------------+
         |
   +-------------------+     +-------------------+
   |  Frontend EC2     | <-- |  Backend EC2      |
   +-------------------+     +-------------------+
                         |
                    MongoDB Atlas
```

---

### 2. Backend Setup (EC2 + Nginx)

```bash
# SSH into backend EC2
ssh -i key.pem ubuntu@<BACKEND_PUBLIC_IP>

# Install Node.js and Nginx
sudo apt update
sudo apt install -y nodejs nginx

# Clone and set up backend
git clone <repo>
cd TravelMemory/backend
npm install

# Create .env file
echo "PORT=3000" > .env
echo "MONGODB_URL=mongodb+srv://..." >> .env

# Start app (use pm2 or nohup)
nohup node index.js &
```

**Nginx Reverse Proxy** (`/etc/nginx/sites-available/travelmemory-backend`):

```nginx
server {
    listen 80;
    server_name api.travelmemory.online;

    location / {
      proxy_pass http://localhost:3000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
}
```
![sites-available_BE](https://github.com/user-attachments/assets/21965dac-6254-47de-a8f5-095d615e47b3)


---

### 3. Frontend Setup (React + Nginx)

```bash
# SSH into frontend EC2
ssh -i key.pem ubuntu@<FRONTEND_PUBLIC_IP>

# Install Nginx and Node.js (for build)
sudo apt install -y nginx nodejs npm

# Clone frontend repo
git clone <repo>
cd travelmemory-frontend

# Update API base URL in src/url.js to:
 export const API_BASE_URL = "https://api.travelmemory.online";

# Build React app
npm install
npm run build

# Copy build to web root (if needed)
sudo cp -r build/* /var/www/html/
```

**Nginx Config** (`/etc/nginx/sites-available/default`):

```nginx
server {
    listen 80;
    server_name travelmemory.online;

    root /var/www/html;
    index index.html;

    location / {
      try_files $uri /index.html;
    }
}
```

---

### 4. AWS Load Balancer (ALB)

- **Create Target Group**: `travelmemory-backend-tg`, port 3000
- **Register** backend EC2(s)
- **Create ALB**: HTTP listener → forwards to target group
- **Set Health Check path**: `/health`

---

### 5. Cloudflare DNS Configuration

![Cloudflare_DNS](https://github.com/user-attachments/assets/c7df4be6-9d2d-4f8b-82e5-3f7aaa6c1b73)


> **Note:** Root domain cannot use CNAME. Use an A record with the frontend EC2 IP.


### 7. Health Check and Final Test

- Visit: `https://travelmemory.online` → Frontend loads
- Visit: `https://api.travelmemory.online/health` → API responds 200 OK
![travelmemory-online](https://github.com/user-attachments/assets/cfe3e7cc-0c8f-44e4-ad4b-10ba1939c110)
![travelmemory-online_BE](https://github.com/user-attachments/assets/08bbe451-12b2-4200-b179-84e616054d36)

---

## Extras

**.env.example**
```env
PORT=3000
MONGODB_URL=mongodb+srv://<username>:<password>@cluster.mongodb.net/travelmemory
```

**src/config.js (Frontend)**
```js
export const API_BASE_URL = "https://api.travelmemory.online";
```
