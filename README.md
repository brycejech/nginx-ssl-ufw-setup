# Nginx, UFW, Let's Encrypt Setup Guide

Guide for setting up Nginx on Ubuntu 16.04 with Universal Firewall (ufw) and SSL with Let's Encrypt


## Install Pre-requisites

```bash
# Add the certbot ppa
sudo add-apt-repository ppa:certbot/certbot

sudo apt-get update

# Install nginx, certbot plugin, and ufw if not installed
sudo apt-get install nginx python-certbot-nginx ufw
```

## Set up Nginx

Edit the default site in sites-available

```bash
sudo nano /etc/nginx/sites-available/default
```

Find the existing `server_name` line and replace it with your domain name. It is important that you use the same server names that you will use with Let's Encrypt. These must match or certbot will not know which server block in the configuration file to update.

```bash
server_name example.com www.example.com;
```

Verify the syntax of the config file with `sudo nginx -t`

If config validation runs without error, reload Nginx with the new config file

```bash
sudo systemctl reload nginx
```

## Set up UFW

Make sure IPV6 is enabled

```bash
sudo nano /etc/default/ufw
```

Make sure the line `IPV6=yes` is uncommented.

Set up strict incoming defaults

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow http, https, and ssh traffic
```bash
sudo ufw allow OpenSSH
# Note that if you are using a non-standard port you can specify a port number
# sudo ufw allow 2222

sudo ufw allow 'Nginx Full'
sudo ufw delete allow 'Nginx HTTP'
```

Check status of ufw with `sudo ufw status`, you should get output similar to this

```bash
Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW       Anywhere
OpenSSH                    ALLOW       Anywhere
Nginx Full (v6)            ALLOW       Anywhere (v6)
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

Make sure ufw is enabled
```bash
sudo ufw enable
```

Can always reset ufw config with `sudo ufw reset` and disable with `sudo ufw disable`

## Set up SSL Cert with certbot

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

If the installation runs successfully, certbot will ask you if you want to redirect all traffic to https or not. You should select the option to redirect all traffic to https.

Certificates have now been registered, installed, and loaded. Try loading your site using https.

### Set up SSL Cert Auto Renewal

Let's Encrypt certs are only valid for 90 days. We need to set up a cronjob to renew our certs automatically. To edit our cronjobs, type `sudo crontab -e`. Add a line to run the certbot renewal:

```bash
15 3 * * * /usr/bin/certbot renew --quiet
```

This will run the renewal daily at 3:15am

## Further SSL hardening

With it's current configuration, we will be capped at a B grade using the [SSL Labs Server Test](https://www.ssllabs.com/ssltest/) due to weak Diffie-Hellman parameters. We are going to genereate a new `dhparam.pem` file and add it to our server block.

```bash
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

This will take a long time to run. Once it's complete, edit your nginx default configuration file in `/etc/nginx/sites-available/default`.

Add the following line to your server block:

```bash
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

We're also going to remove support for TLSv1.0. Add the following lines after the `listen 443 ssl;` directive:

```bash
ssl_protocols 	TLSv1.1 TLSv1.2;
```

Check that you don't have any syntax errors with `sudo nginx -t`.

If all goes well, reload nginx with the following:

```bash
sudo systemctl reload nginx
```


## Enable Gzip

By default Nginx only compresses html files. Edit the file at `/etc/nginx/nginx.conf`.

```bash
sudo nano /etc/nginx/nginx.conf
```

Find the `gzip` section and edit it to look like this:

```bash
gzip on;
gzip_disable "msie6";

gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_min_length 256;
gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/x-icon;
```

Verify proper syntax with `sudo nginx -t`. If all goes well reload nginx conf with `sudo systemctl reload nginx` and you are good to go ;)


This guide was put together from several Digital Ocean guides. The can be found here:
[Secure Nginx with Let's Encrypt](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)
[Set up a Firewall with UFW on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-16-04)
[Add gzip to Nginx on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-16-04)
