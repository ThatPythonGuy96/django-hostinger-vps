## Django + Gunicorn as a systemd service (end-to-end)

### check user permission
sudo -l -U <username>

sudo whoami

### Allow the user to run admin commands (sudo) 
usermod -aG sudo <username>

### Give the user ownership of your app files
chown -R <username>:<username> /opt/django-app

### 1) SSH into the VPS
ssh root@77.37.120.77

### 2) Clone Your GitHub repo and Go to your project folder
cd /usr/local/lsws/Example/html
mkdir -p /opt/django-app
cd /opt/django-app
git clone <YOUR_GIT_REPO_URL> app
cd app

apt update
apt install libpq-dev python3-dev gcc -y

### 3) Create/activate a virtual environment
python3 -m venv venv
source venv/bin/activate

### 4) Install dependencies (including Gunicorn)
sudo bash build.sh
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn

### 5) Confirm the Django app module you’ll run
### Typical is `yourproject.wsgi:application`. In your case it’s:
python -c "import stashorra.wsgi; print('OK')"

### 6) Quick manual Gunicorn test (before systemd)
/opt/django-app/app/venv/bin/gunicorn --workers 2 --bind 127.0.0.1:8001 stashorra.wsgi:application
### Stop it with `Ctrl+C`.

### 7) Create the systemd service file
sudo nano /etc/systemd/system/stashorra-gunicorn.service 
'EOF'
[Unit]
Description=Gunicorn for Django (stashorra)
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/django-app/app
Environment="PATH=/opt/django-app/app/venv/bin"
ExecStart=/opt/django-app/app/venv/bin/gunicorn --workers 2 --bind 127.0.0.1:8001 stashorra.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

### 8) Reload systemd, enable, and start
sudo systemctl daemon-reload
sudo systemctl enable --now stashorra-gunicorn

### 9) Check status + logs
sudo systemctl status stashorra-gunicorn --no-pager
sudo journalctl -u stashorra-gunicorn -n 50 --no-pager

### 10) Local HTTP check (from the VPS)
curl -I http://127.0.0.1:8001

```
    HTTP/1.1 200 OK
    Server: gunicorn
    Date: Tue, 10 Feb 2026 21:31:23 GMT
    Connection: close
    Content-Type: text/html; charset=utf-8
    Cross-Origin-Opener-Policy: unsafe-none
    Allow: GET, HEAD, OPTIONS
    X-Frame-Options: DENY
    Vary: Cookie, origin
    Content-Length: 4630
    X-Content-Type-Options: nosniff
    Referrer-Policy: same-origin
    Set-Cookie: csrftoken=S1Ny9qNXDY5rLrrQUqRTuzWVDzSftQb7; expires=Tue, 09 Feb 2027 21:31:23 GMT; Max-Age=31449600; Path=/; SameSite=None; Secure
```
## Common fixes
### A) `status=217/USER`
### The `User=` doesn’t exist. Use an existing user like `www-data` (or create one), then restart the service.

### B) `status=203/EXEC`
### The `ExecStart=` path is wrong or `gunicorn` isn’t installed in that venv. Confirm:
ls -l /opt/django-app/app/venv/bin/gunicorn
### Then restart:
sudo systemctl restart stashorra-gunicorn

<!-- Give user <www-data> permission to write -->
chown -R www-data:www-data /opt/django-app/app

### OpenLiteSpeed WebAdmin
ufw allow 7080/tcp
ufw reload

WebAdmin: https://77.37.120.77:7080


### When gitbut code is updated
cd /opt/django-app/app
git pull
sudo bash build.sh
sudo systemctl daemon-reload
### restart gunicorn
sudo systemctl restart stashorra-gunicorn
### restart OpenLiteSpeed (Optional)
sudo systemctl restart lsws

### OpenLiteSpeed WebAdmin

OpenLiteSpeed WebAdmin → set up proxy

Go to WebAdmin: https://77.37.120.77:7080 (login as admin)
### Reset Username and Password
/usr/local/lsws/admin/misc/admpass.sh
###
Virtual Hosts → select your vhost (or create one for your domain)
External App → Add
Type: Web Server
Name: stashorra-gunicorn
Address: 127.0.0.1:8001
Context → Add
Type: Proxy
URI: /
Destination: stashorra-gunicorn

Restart OpenLiteSpeed

### Start Gunicorn
cd /opt/django-app/app
/opt/django-app/venv/bin/gunicorn --bind 127.0.0.1:8001 stashorra.wsgi:application
