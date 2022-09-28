# Listmonk Setup

## AWS configuration

```text
EC2 image, Ubuntu 22.04, t3a.small
Extra Volume, 14gb, mounted to EC2 image
Aurora RDS, Postgres 14.3, db.t4g.medium
```

### Security Group Settings

- Security Group which contains EC2 image, RDS and Redis services
  - Port 80: 0.0.0.0/0 (HTTP)
  - Port 443: 0.0.0.0/0 (HTTPS)
  - Port 5432: SG only (Postgres)

## Collecting additional information

This setup depends on quite a number of secrets in order to automate SSL renewal, domain names being updated, providing a secure login and email being delivered.

In the file `dot_env` there are three secrets set. Furthermore in the `secrets` directory these variables need to be stored as files as well.

### Auth0

- The Web Application's Client ID, stored as a value in the secret `secrets/traefik_forward_auth` file for the key `providers.generic-oauth.client-id`
- The Web Application's Client Secret, stored as a value in the secret `secrets/traefik_forward_auth` file for the key `providers.generic-oauth.client-secret`

### AWS

- AWS SES email user name, stored in .env as value for `SMTP_USER`
- AWS SES email password, stored in .env as value for `SMTP_PASS`

### Cloudflare

- Zone ID where "DOMAINNAME0" lives, stored in .env as value for `CLOUDFLARE_ZONEID`
- An API token with DNS:Edit rights on the above Zone ID, stored in .env as value for `CLOUDFLARE_DNS_API_TOKEN` **and** as of the content of the `secrets/cf_dns_token` file.
- An API token with Zone:View rights on the above Zone ID, stored as the content of the `secrets/cf_zone_token` file.

## Instance setup (as ubuntu user)

### Mount the extra EBS volume

```bash
sudo mkfs -t ext4 /dev/nvme1n1
sudo mkdir -p /app/pretalx
echo "/dev/nvme1n1 /app/pretalx ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

### Install dependencies

```bash
sudo apt update; sudo apt dist-upgrade; sudo reboot
sudo apt install -y net-tools ca-certificates curl gnupg lsb-release
```

### Setup Docker APT repo & Install Docker

```bash
 echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 sudo apt update; sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Setup Listmonk user

```bash
sudo adduser listmonk --disabled-password --home /app/listmonk
sudo usermod -aG docker listmonk
sudo chown -R listmonk:listmonk /app/listmonk
```

### Login as listmonk user

`sudo su - listmonk`

## Instance Setup (as listmonk user)

### Populating secrets / sensitive data

```bash
mkdir secrets
cd secrets; touch cf_dns_token cf_zone_token traefik_forward_auth; cd ..
```

Replace the content of:

- `secrets/cf_dns_token`
- `secrets/cf_zone_token`
- `secrets/traefik_forward_auth`
- `.env`

with the earlier collected additional information.

### Putting stateful directories & files in place

```bash
cd ~; mkdir -p data/docker-gc data/listmonk/{static,uploads} data/traefik2/{acme,logs,rules} data/watchtower
cp listmonk-config/config.toml data/listmonk/config.toml
cp -v traefik2-rules/*yml data/traefik2/rules/
```

### Copying in docker-compose

```bash
cp compose/docker-compose ~/
```

### Start up Docker

```bash
docker compose up -d; docker compose logs --follow
```

Watch output to see for errors regarding Lets Encrypt or communicating with Cloudflare.
