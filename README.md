# Deployment

## For Testing Purposes

If you just want to test if Modern MISP fits you, you can setup a server in about 30 minutes.
Make sure you have a valid API token for your current MISP. 
We hope to provide packaged versions soon.

### Prepare database host

Create a new database, create a new user and grant permissions on the new database.

```mysql
CREATE DATABASE modern_misp;
GRANT ALL PRIVILEGES ON modern_misp.* TO 'modern_misp_user'@'%' IDENTIFIED BY 'new password'
```

Use your current misp database and copy it to the new database

```shell
mysqldump misp > misp.sql
mysql modern_misp < misp.sql
```

### Setup modern misp

On a clean debian 12, install needed packages

```shell
apt-get install redis nginx python3-venv npm
```

Create a new user.
Switch to the user and clone the repositories (frontend,api,worker).

```
adduser mmisp
sudo -iu mmisp
git clone https://github.com/Modern-MISP/worker.git
git clone https://github.com/Modern-MISP/frontend.git
git clone https://github.com/Modern-MISP/api.git
```

#### Worker setup

As User, cd into your checkout.
Create a new venv, activate it and install the worker.

```shell
cd worker
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
deactivate
```

As root, create a systemd service (`/etc/systemd/system/mmisp-worker.service`):

```
[Unit]
Description=mmisp worker

[Service]
Restart=on-failure
TimeoutStopSec=70
Environment=MMISP_API_KEY=random string
Environment=MMISP_API_PORT=8001
Environment=MMISP_API_HOST=127.0.0.1
WorkingDirectory=<path to checkout>
ExecStart=<path to checkout>/.venv/bin/mmisp-worker
Type=simple
User=<user>

[Install]
WantedBy=multi-user.target
```

#### API Setup

As User, cd into your checkout.
Create a new venv, activate it and install the worker.

```
cd api
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
deactivate
```

As root, create a systemd service (`/etc/systemd/system/mmisp-api.service`):

```
[Unit]
Description=mmisp api

[Service]
Restart=on-failure
TimeoutStopSec=70
Environment=DATABASE_URL=mysql+aiomysql://<DB_USER>:<DB_PASSWORD>@<DB_HOST>:3306/<DB_NAME>
WorkingDirectory=<path to checkout>
ExecStart=<path to checkout>/.venv/bin/uvicorn mmisp.api.main:app --port=8000
Type=simple
User=<user>

[Install]
WantedBy=multi-user.target
```

#### Frontend Setup

As user, build the install the dependencies and build the frontend.

```
npm install
npm run build
```

As root, copy the build an appropriate location:
Make sure you don't forget the trailing slash on `build/`

```
mkdir /var/www/mmisp
rsync -r <frontend checkout>/build/ /var/www/mmisp/
```

Place a config.yaml file in `/var/www/misp`

```yaml
---

MISP_API_ENDPOINT: https://<api server name>
```

#### Nginx Setup (Reverse Proxy)

Adapt the following nginx configuration.

```nginx
server {                                                                    
        listen 80 default_server;                                           
        listen [::]:80 default_server;           
                                      
        server_name _;

        return 301 https://$host$request_uri;
} 

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name <server name for frontend>;

        ssl_certificate <certificate file>;
        ssl_certificate_key <key file>;

        root /var/www/mmisp;
        index index.html;

        location / {
                try_files $uri $uri/ /index.html =404;
        }
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name <server name for api>;

        ssl_certificate <certificate file>;
        ssl_certificate_key <key file>;


        location / {
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 300s;
                add_header Front-End-Https on;
                proxy_pass http://127.0.0.1:8000;
        }
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name <server name for worker>;

        ssl_certificate <certificate file>;
        ssl_certificate_key <key file>;

        location / {
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 300s;
                add_header Front-End-Https on;
                proxy_pass http://127.0.0.1:8001;
        }
}
```

### Starting the services

```
systemctl daemon-reload
systemctl start mmisp-api mmisp-worker
systemctl reload nginx
```

## Using Modern MISP Frontend with MISP API

You need to tweak some settings for cors to work. Alter your `/etc/apache/sites-available/misp-ssl.conf` to include the following
settings inside the virutal host file:

```
Header unset Access-Control-Allow-Origin
Header unset Access-Control-Allow-Methods
Header always set Access-Control-Allow-Origin "*"
Header always set Access-Control-Allow-Headers "Authorization, Content-Type"
Header always set Access-Control-Allow-Methods "GET, POST, PUT, PATCH, DELETE"
Header always set Access-Control-Expose-Headers "Content-Security-Policy, Location"
Header always set Access-Control-Max-Age "600"

RewriteEngine On
RewriteCond %{REQUEST_METHOD} OPTIONS
RewriteRule ^(.*)$ $1 [R=200,L]
```

Change some settings in the MISP Database:

```
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting Security.allow_cors true
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting Security.cors_origins '*'
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting Plugin.Workflow_enable true
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting Security.check_sec_fetch_site_header false
```
