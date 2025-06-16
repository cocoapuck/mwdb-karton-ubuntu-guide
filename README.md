# Install MWDB and Karton on Ubuntu

This setup is tested on Ubuntu 24.04 LTS and is intended for on-prem lab environments and should not be used on servers exposed to the internet. Review of hardening and security is needed.

All services should run as a nonpriviliged user. In this guide this user is referred to "someadmin". Change this to your actual user.

TODO:
- Env-file or variables for creds to Minio, Postgres etc.
- TLS-Cert for web-servers
- Hardening
- Guide for custom Karton service (eg. Clamav, checksec, strings) based on this: https://github.com/CERT-Polska/karton-playground


## Useful information and guides

MWDB documentation:
https://mwdb.readthedocs.io/en/latest/setup-and-configuration.html

Training documentation:
https://training-mwdb.readthedocs.io/en/latest/

Karton documentation:
https://karton-core.readthedocs.io/en/latest/


## Install a minimal Ubuntu Server 24.04 LTS

Enable SSH-server in the install wizard.
Keep everything else as default.

`sudo apt update`

`sudo apt upgrade -y`

## Install basic tools and utilities

`sudo apt install nano vim curl git`

## Install MWDB/Karton/Redis/Minio/Nginx dependencies

`sudo apt install gcc libfuzzy-dev python3-dev python3-venv postgresql postgresql-client postgresql-common libmagic1 lsb-release gpg nginx -y`

## Create database and user for MWDB
```
sudo -u postgres psql

postgres=# CREATE DATABASE mwdb;
postgres=# CREATE USER mwdb WITH PASSWORD 'mwdb';
postgres=# GRANT ALL PRIVILEGES ON DATABASE mwdb TO mwdb;
postgres=# ALTER DATABASE mwdb OWNER TO mwdb;
```

#### Postgres connection string
`postgresql://username:password@127.0.0.1:5432/database`

Eg. postgresql://mwdb:mwdb@127.0.0.1:5432/mwdb


## Setup Python virtual environments

The MWDB and Karton servers will be installed in the /opt directory so run the installation as `sudo su`. Later we will change the owner of the directories.
##### Python venv for MWDB
`mkdir /opt/mwdb`

`cd /opt/mwdb`

`python3 -m venv .mwdb`

`source .mwdb/bin/activate`

`pip install mwdb-core`

`chown someadmin:someadmin -R /opt/mwdb`

`deactivate`


##### Python venv for KARTON
`mkdir /opt/karton`

`cd /opt/karton`

`python3 -m venv .karton`

`source .karton/bin/activate`

`pip install karton-core karton-mwdb-reporter karton-classifier karton-asciimagic karton-dashboard`

`chown someadmin:someadmin -R /opt/karton`

`deactivate`

Exit sudo

`exit`


TODO:
- karton-yaramatcher
- karton-unpacker
- karton-config-extractor
- karton-autoit-ripper
- karton-archive-extractor
- karton-misp-pusher


## Install Redis (required for Karton)

Add Redis repo to sources
```
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
```

Install Redis

`sudo apt update`

`sudo apt install redis`


## Install Minio (Required for Karton)

Create directories

`sudo mkdir /opt/minio /opt/minio/bin /opt/minio/data`

Download binary

`sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio -O /opt/minio/bin/minio`

Make executable and change owner

`sudo chmod +x /opt/minio/bin/minio`

`sudo chown someadmin:someadmin -R /opt/minio`

Create a config file for Minio

`nano /opt/minio/minio.conf`

Content of mino.conf
```
MINIO_VOLUMES="/opt/minio/data"
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER="miniokarton"
MINIO_ROOT_PASSWORD="miniokarton"
```

##### Configure Minio to run as a systemd service

Create a service file

`sudo nano /usr/lib/systemd/system/minio.service`

Content of minio.service
```
[Unit]
Description=Minio
Documentation=https://docs.minio.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/opt/minio/bin/minio

[Service]
WorkingDirectory=/opt/minio

User=someadmin
Group=someadmin

#PermissionsStartOnly=true

EnvironmentFile=-/opt/minio/minio.conf
ExecStartPre=/bin/bash -c "[ -n \"${MINIO_VOLUMES}\" ] || echo \"Variable MINIO_VOLUMES not set in /opt/minio/minio.conf\""

ExecStart=/opt/minio/bin/minio server $MINIO_OPTS $MINIO_VOLUMES 
StandardOutput=journal
StandardError=inherit
# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536
# Disable timeout logic and wait until process is stopped
TimeoutStopSec=0
# SIGTERM signal is used to stop Minio
KillSignal=SIGTERM
SendSIGKILL=no
SuccessExitStatus=0
[Install]
WantedBy=multi-user.target
```

Reload systemd deamon

`sudo systemctl daemon-reload`

Enable and start the service
```
sudo systemctl enable minio
sudo systemctl start minio
```

Login to minio-web ui (http://<your_ip_address:9001) and create a new bucket "karton".


## Configure MWDB and Karton

#### MWDB

`cd /opt/mwdb`

`source .mwdb/bin/activate`

`mwdb-core configure`

Select "3. Current directory"
Add the Postgres-url from earlier: postgresql://mwdb:mwdb@127.0.0.1:5432/mwdb

Accept defaults for upload-path and URL.

Add a password for the admin-user for MWDB. This will be used to login to the web ui.

Enable MWDB-Karton integration by setting `enable_karton = 1` option in `/opt/mwdb/mwdb.ini`.
Provide a `karton.ini` file by setting `config_path = /etc/karton/karton.ini`.

```
[mwdb]
...
enable_karton = 1

[karton]
config_path = /opt/karton/karton.ini
```

##### Configure MWDB and Karton to run as a systemd service

Create service file

`sudo nano /usr/lib/systemd/system/mwdb.service`

Contents of mwdb.service
```
[Unit]
Description=MWDB-Core run app
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/mwdb
Environment="PATH=/opt/mwdb/.mwdb/bin"
ExecStart=/opt/mwdb/.mwdb/bin/mwdb-core run
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Reload systemd deamon

`sudo systemctl daemon-reload`

Enable and start the service
```
sudo systemctl enable mwdb
sudo systemctl start mwdb
```


## Install and configure NGINX as a proxy for MWDB web ui

Remove default config and create a new config for MWDB

`sudo rm /etc/nginx/sites-enabled/default`

`sudo nano /etc/nginx/sites-enabled/mwdb`

Contents of /etc/nginx/sites-enabled/mwdb

```
server {
    listen 80;
    server_name = mwdb;
    location / {
        proxy_pass http://localhost:5000;
    }
}
```

Reload Nginx

`sudo systemctl reload nginx`


## Configure MWDB to integrate with Karton

Login to the MWDB webinterface and go to Settings -> User -> Add user. Add a new user called `Karton`.

Go to the Settings -> Access control and give karton all the required capabilities:
- adding_tags
- adding_comments
- adding_parents
- adding_all_attributes
- adding_files
- adding_configs
- adding_blobs
- unlimited_requests
- karton_assign

Open to the `karton` account details and click on `Manage API keys` action to create an API key for this account. Click `Issue new API key` to create the key. Copy the key to paste in config file.

Create a configuration file for Karton

`nano /opt/karton/karton.ini`

Contents of karton.ini
```
[s3]
secret_key = miniokarton
access_key = miniokarton
address = http://localhost:9000
bucket = karton

[redis]
host=localhost
port=6379

[mwdb]
api_url = http://127.0.0.1/api/
api_key = <paste the key here...>
```


##### Configure the Karton tools to run as systemd services

Create service files

```
sudo nano /usr/lib/systemd/system/karton-system.service
sudo nano /usr/lib/systemd/system/karton-mwdb-reporter.service
sudo nano /usr/lib/systemd/system/karton-classifier.service
sudo nano /usr/lib/systemd/system/karton-asciimagic.service
sudo nano /usr/lib/systemd/system/karton-dashboard.service
```

Contents of karton-system.service
```
[Unit]
Description=Karton System
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/karton
Environment="PATH=/opt/karton/.karton/bin"
ExecStart=/opt/karton/.karton/bin/karton-system
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Contents of karton-mwdb-reporter.service
```
[Unit]
Description=Karton MWDB Reporter
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/karton
Environment="PATH=/opt/karton/.karton/bin"
ExecStart=/opt/karton/.karton/bin/karton-mwdb-reporter
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Contents of karton-classifier.service
```
[Unit]
Description=Karton Classifier
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/karton
Environment="PATH=/opt/karton/.karton/bin"
ExecStart=/opt/karton/.karton/bin/karton-classifier
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Contents of karton-asciimagic.service
```
[Unit]
Description=Karton ASCII MAgic
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/karton
Environment="PATH=/opt/karton/.karton/bin"
ExecStart=/opt/karton/.karton/bin/karton-asciimagic
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Contents of karton-dashboard.service
```
[Unit]
Description=Karton Dashboard
After=network.target

[Service]
User=someadmin
Group=someadmin
WorkingDirectory=/opt/karton
Environment="PATH=/opt/karton/.karton/bin"
ExecStart=/opt/karton/.karton/bin/karton-dashboard run -h 0.0.0.0 -p 5001
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```


Reload systemd deamon

`sudo systemctl daemon-reload`

Enable and start the services

```
sudo systemctl enable karton-system karton-mwdb-reporter karton-classifier karton-asciimagic karton-dashboard

sudo systemctl start karton-system karton-mwdb-reporter karton-classifier karton-asciimagic karton-dashboard
```

The Karton dashboard should be accessible on http://<mwdb_url>:5001


















   
   
