# Pretalx Setup

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

## Instance setup (as ubuntu user)

### Mount the extra EBS volume

```bash
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir -p /app/pretalx
echo "/dev/nvme1n1 /app/pretalx ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

### Symlink directory to root volume

```bash
cd /etc; sudo ln -s /app/pretalx pretalx
sudo touch /etc/pretalx/pretalx.cfg
sudo chmod 0600 /etc/pretalx/pretalx.cfg
sudo adduser pretalx --disabled-password --home /app/pretalx
sudo chown -R pretalx:pretalx /etc/pretalx/
```

### Install dependencies

```bash
sudo apt update; sudo apt dist-upgrade; sudo reboot
sudo apt install python3 python-is-python3 postgresql-client build-essential libssl-dev python3-dev gettext python3-pip certbot python3-certbot-nginx
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
create role pretalx with createdb login password '<SOMERANDOLONGPASSWORD>';
grant pretalx to postgres;
create database pretalx_db with owner pretalx;
quit;
```

#### Check setup by logging in with pretalx db user

```bash
psql -d pretalx_db  -U pretalx -h FULL_HOST_NAME_OF_RDS_INSTANCE -p 5432
```

### NGINX and certbot Configuration

```bash
sudo cp -v nginx-conf/nginx.conf /etc/nginx/nginx.conf
sudo cp -v nginx-conf/start-talx.devops.foundation.conf /etc/nginx/conf.d/talx.devops.foundation.conf
sudo nginx -t
sudo systemctl start nginx
sudo systemctl enable nginx
sudo certbot --nginx -d talx.devops.foundation
sudo cp -v nginx-conf/finish-talx.devops.foundation.conf /etc/nginx/conf.d/talx.devops.foundation.conf
sudo nginx -t
sudo nginx restart
```

### pretalx setup

***Note: You will need to double check the settings in `pretalx-conf/pretalx.cfg`.***

```bash
sudo cp -v pretalx-conf/pretalx.cfg /etc/pretalx/pretalx.cfg
sudo chown pretalx:pretalx /etc/pretalx/pretalx.cfg
```

#### Login as the pretalx user

`sudo su - pretalx`

#### as pretalx user

pip install --user -U pip setuptools wheel gunicorn psycopg2-binary
mkdir -p /app/pretalx/data/media
mkdir -p /app/pretalx/static
pip install --user --upgrade-strategy eager -U pretalx[redis]
python -m pretalx migrate
python -m pretalx rebuild
python -m pretalx init

### Start 'er up

```bash
sudo cp -v systemd-files/pretalx* /etc/systemd/system
sudo systemctl start pretalx-web.service pretalx-worker.service pretalx-periodic.timer
sudo systemctl enable pretalx-web.service pretalx-worker.service pretalx-periodic.timer
```
