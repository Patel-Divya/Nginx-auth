```text
Internet
   ↓
Nginx (auth + reverse proxy)
   ↓
Docker Kibana (5601 exposed BUT controlled via firewall/Nginx)
```

## FILES TO CREATE/MODIFY

### 1. Nginx site config

```
/etc/nginx/sites-available/kibana
```

### 2. Enabled symlink

```
/etc/nginx/sites-enabled/kibana
```

### 3. Authentication file (multi-user)

```
/etc/nginx/.htpasswd
```

### 4. (optional) firewall rule

```
ufw or iptables
```

---

## STEP 1 — INSTALL REQUIRED PACKAGES

```bash
sudo apt update
sudo apt install nginx apache2-utils -y
```

---

## STEP 2 — CREATE MULTI-USER LOGIN FILE

We create a password file that Nginx will use.

### Create first user:

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

👉 This creates file + first user

---

### Add more users:

```bash
sudo htpasswd /etc/nginx/.htpasswd user1
sudo htpasswd /etc/nginx/.htpasswd user2
```


---

## STEP 3 — CREATE NGINX CONFIG

Create file:

```bash
sudo nano /etc/nginx/sites-available/kibana
```

---

### Paste this config:

```nginx
server {
    listen 80;
    server_name _;

    # AUTHENTICATION (LOGIN SCREEN)
    auth_basic "Kibana Secure Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # PROXY TO KIBANA
    location / {
        proxy_pass http://127.0.0.1:5601;

        # Required for Kibana WebSockets (VERY IMPORTANT)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Forward headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## EXPLANATION OF CONFIG

### auth_basic

* Forces browser login popup
* Blocks access before Kibana loads

---

### auth_basic_user_file

* Points to `/etc/nginx/.htpasswd`
* Contains all users

---

### proxy_pass

```text
http://127.0.0.1:5601
```

* Sends traffic to Kibana
* Using localhost ensures safer routing

---

### WebSocket headers

Required because Kibana uses real-time features:

* dashboards
* Discover
* logs streaming

Without this:
Kibana breaks or loads blank pages

---

## STEP 4 — ENABLE SITE

```bash
sudo ln -s /etc/nginx/sites-available/kibana /etc/nginx/sites-enabled/
```

---

## STEP 5 — TEST CONFIG

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

---

## STEP 6 — RESTART NGINX

```bash
sudo systemctl restart nginx
```

---

## STEP 7 — TEST ACCESS

Open in browser:

```text
http://YOUR_SERVER_IP/
```

You should see:

1. Login popup (Nginx auth)
2. After login → Kibana dashboard loads

---

## STEP 8 — BLOCK DIRECT ACCESS TO 5601 (CRITICAL)

Right now Kibana is still exposed by Docker.

#### Check:

```bash
curl http://YOUR_IP:5601
```

---

### OPTION A

Block via firewall:

```bash
sudo ufw allow 80
sudo ufw deny 5601
```

---

### OPTION B 

```bash
sudo iptables -A INPUT -p tcp --dport 5601 -j DROP
```


