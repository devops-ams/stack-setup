# Pretix Instance Setup

## AWS configuration

```text
EC2 image, Ubuntu 22.04, t3a.small
Extra Volume, 14gb, mounted to EC2 image
Aurora RDS, Postgres 14.3, db.t4g.medium
ElastiCache, Redis 6.2.6, cache.t4g.micro
```

### Security Group Settings

- Security Group which contains EC2 image, RDS and Redis services
  - Port 80: 0.0.0.0/0 (HTTP)
  - Port 443: 0.0.0.0/0 (HTTPS)
  - Port 5432: SG only (Postgres)
  - Port 6379: SG only (Redis)
  - Port 8345: SG only (Pretix)

## Instance setup (as ubuntu user)

### Mount the extra EBS volume

```bash
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir -p /app/pretix
echo "/dev/nvme1n1 /app/pretix ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

### Symlink directory to root volume

```bash
cd /etc; sudo ln -s /app/pretix pretix
sudo touch /etc/pretix/pretix.cfg
sudo chmod 0600 /etc/pretix/pretix.cfg
sudo adduser pretix --disabled-password --home /app/pretix
sudo chown -R pretix:pretix /etc/pretix/
```

### Install dependencies

```bash
sudo apt update; sudo apt dist-upgrade; sudo reboot
sudo apt install -y git build-essential python3-dev libxml2-dev libxslt1-dev libffi-dev zlib1g-dev libssl-dev gettext libpq-dev libjpeg-dev libopenjp2-7-dev python3-virtualenv python3-pip postgresql-client python3.10-venv redis-tools python-is-python3 wget gnupg2 ca-certificates lsb-release ubuntu-keyring software-properties-common nodejs certbot python3-certbot-nginx
```

### Install NPM

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
```

### Setup apt repo and install nginx

```bash
sudo curl --retry 3 --retry-delay 10 -sSL https://nginx.org/keys/nginx_signing.key | sudo gpg --dearmor --output /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" | sudo tee /etc/apt/sources.list.d/nginx-mainline.list
echo "deb-src [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" | sudo tee -a /etc/apt/sources.list.d/nginx-mainline.list
sudo apt update; sudo apt install nginx
```

### Configure Postgres

#### Connect to AuroraDB Postgres instance

```bash
psql  -U postgres -h FULL_HOST_NAME_OF_RDS_INSTANCE -p 5432
```

#### Create account

```pgsql
create role pretix with createdb login password '<SOMERANDOLONGPASSWORD>';
grant pretix to postgres;
create database pretix_db with owner pretix;
quit;
```

#### Check setup by logging in with pretix db user

```bash
psql -d pretix_db  -U pretix -h FULL_HOST_NAME_OF_RDS_INSTANCE -p 5432
```

### NGINX and certbot Configuration

```bash
sudo cp -v nginx-conf/nginx.conf /etc/nginx/nginx.conf
sudo cp -v nginx-conf/start-tix.devops.foundation.conf /etc/nginx/conf.d/tix.devops.foundation.conf
sudo nginx -t
sudo systemctl start nginx
sudo systemctl enable nginx
sudo certbot --nginx -d tix.devops.foundation
sudo cp -v nginx-conf/finish-tix.devops.foundation.conf /etc/nginx/conf.d/tix.devops.foundation.conf
sudo nginx -t
sudo nginx restart
```

### Pretix setup

***Note: You will need to double check the settings in `pretix-conf/pretix.cfg`.***

```bash
sudo cp -v pretix-conf/pretix.cfg /etc/pretix/pretix.cfg
sudo chown pretix:pretix /etc/pretix/pretix.cfg
```

#### Login as the pretix user

`sudo su - pretix`

#### as pretix user

```bash
python3 -m venv /app/pretix/venv
source /app/pretix/venv/bin/activate
pip3 install -U pip setuptools wheel
pip3 install pretix gunicorn pylibmc
mkdir -p /app/pretix/data/media
python -m pretix migrate
python -m pretix rebuild
```

### Start 'er up

```bash
sudo cp -v systemd-files/pretix* /etc/systemd/system
sudo systemctl start pretix-web.service pretix-worker.service pretix-periodic.timer
sudo systemctl enable pretix-web.service pretix-worker.service pretix-periodic.timer
```
