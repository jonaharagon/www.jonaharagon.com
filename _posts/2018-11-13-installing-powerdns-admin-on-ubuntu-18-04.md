---
layout: post
title: "Installing PowerDNS-Admin on Ubuntu 18.04"
cover: "/assets/img/2018-11-13-installing-powerdns-admin-on-ubuntu-18-04/cover.png"
---

Now that you have [PowerDNS installed](https://jonaharagon.com/setting-up-powerdns-with-mysql-replication-on-ubuntu-18-04/) on your servers, you may be wondering how you and your users can create zones on your servers. Well, [PowerDNS-Admin](https://github.com/ngoduykhanh/PowerDNS-Admin) is a web-based control panel that makes managing PowerDNS a breeze. It has support for multiple accounts with varying privileges, domain-based user access management, support for a variety of backends for user authentication (LDAP, SAML, etc.), and a whole lot more.

We're going to be setting this up on the master DNS server we created in a previous article, but if you choose to set this tool up remotely or on a preexisting server that should work as well with a few minor tweaks for your config. Essentially, this tool interfaces with the PowerDNS API that's built in to the `pdns-server` software, so it should work with any PowerDNS server with a MySQL backend.

## Prerequisites

You should have PowerDNS installed already, with the following changes made to your `/etc/powerdns/pdns.conf` file:

```
api=yes
# Replace <RANDOM STRING> with a randomly generated key for API access
api-key=<RANDOM STRING>
```

We did this already in the previous post.

If you want to configure Let's Encrypt later (which this guide will do), you will also need a publicly accessible domain pointed to your master server.

## Prepare a database

Before we can install PowerDNS-Admin, we'll need to setup a MySQL database for it to use and store its own information. You generally shouldn't use the same database as PowerDNS uses, so we'll create a new one now.

Open MySQL as root:

```
mysql -u root
```

And create a new database:

```
CREATE DATABASE powerdnsadmin CHARACTER SET utf8 COLLATE utf8_general_ci;
```

Then create a new user, flush the privilege table, and quit:

```
GRANT ALL PRIVILEGES ON powerdnsadmin.* TO 'pdnsadminuser'@'%' IDENTIFIED BY '<YOUR PASSWORD HERE>';
FLUSH PRIVILEGES;
quit;
```

Replacing, of course, `<YOUR PASSWORD HERE>` with a new random password.
Installing Required Packages

PowerDNS-Admin requires some software beforehand, so we can build it. Start with installing the Python 3 development package:

```
sudo apt-get install python3-dev
```

And then some additional packages we'll need just to install the rest of the required files:

```
sudo apt-get install -y libmysqlclient-dev libsasl2-dev libldap2-dev libssl-dev libxml2-dev libxslt1-dev libxmlsec1-dev libffi-dev pkg-config
```

As an aside, we're installing `libmysqlclient-dev` because we previously setup PowerDNS with MySQL. If you chose to use MariaDB or PostgreSQL for your setup those backends will work as well, but you'll need to install the appropriate package for your install.

Finally, we'll need to install yarn to build the asset files:

```
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update -y
sudo apt-get install -y yarn
```

These commands add the yarn repository to your system, refreshes your package list, and complete the install.

## Downloading PowerDNS-Admin

We're going to install PowerDNS-Admin to `/opt/web/powerdns-admin` just to keep things simple, but you can adjust that directory to your local web application directory without issue, if you have another preference.

Clone the directory:

```
git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git /opt/web/powerdns-admin
```

If you don't have `git` installed already, you may have to `sudo apt install git`.

Now, change to that new directory:

```
cd /opt/web/powerdns-admin
virtualenv -p python3 flask
```

Next, activate a python3 development environment, and begin installing the libraries required by PowerDNS-Admin:

```
. ./flask/bin/activate
pip install -r requirements.txt
```

## Installation

Now, we'll need to copy the configuration template to a new configuration file:

```
cp /opt/web/powerdns-admin/config_template.py /opt/web/powerdns-admin/config.py
```

Now, edit the file, we'll need to make the following changes:

```
nano /opt/web/powerdns-admin/config.py
```

Specifically, we want to modify the database variables to match our configuration:

```
SQLA_DB_USER = 'pdnsadminuser'
SQLA_DB_PASSWORD = '<YOUR PASSWORD HERE>'
SQLA_DB_HOST = '127.0.0.1'
SQLA_DB_NAME = 'powerdnsadmin'
SQLALCHEMY_TRACK_MODIFICATIONS = True
```

Replacing `<YOUR PASSWORD HERE>` with the password we created at the beginning of this article. Don't change anything else in this file, the port and bind address can stay as they are, because we'll be throwing this entire application behind Nginx for public access in just a bit.

Now, that your configuration is all set, we can create the database schema:

```
export FLASK_APP=app/__init__.py
flask db upgrade
```

Generate asset files:

```
yarn install --pure-lockfile
flask assets build
```

And that's it! We should now be able to run the application. Now, we'll put this behind an Nginx reverse proxy, and setup a systemd service for production use.

## Setting Up Systemd

Create a new service file:

```
/etc/systemd/system/powerdns-admin.service
```

And paste in the following information:

```
[Unit]
Description=PowerDNS-Admin
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=/opt/web/powerdns-admin
ExecStart=/opt/web/powerdns-admin/flask/bin/gunicorn --workers 2 --bind unix:/opt/web/powerdns-admin/powerdns-admin.sock app:app

[Install]
WantedBy=multi-user.target
```

Now, reload systemd to apply your changes:

```
sudo systemctl daemon-reload
```

And start the application:

```
sudo systemctl start powerdns-admin
```

Finally, if you don't run into any errors, enable the service, so it starts at boot:

```
sudo systemctl enable powerdns-admin
```

## Creating an Nginx Site

If you don't have Nginx installed yet, simply run this command to install it now:

```
sudo apt install nginx
```

Now, create a new configuration file for your website:

```
sudo nano /etc/nginx/sites-enabled/powerdns-admin.conf
```

And paste in the following:

```
server {
  listen *:80;
  server_name               <YOUR DOMAIN NAME HERE>;

  index                     index.html index.htm index.php;
  root                      /opt/web/powerdns-admin;
  access_log                /var/log/nginx/powerdns-admin.local.access.log combined;
  error_log                 /var/log/nginx/powerdns-admin.local.error.log;

  client_max_body_size              10m;
  client_body_buffer_size           128k;
  proxy_redirect                    off;
  proxy_connect_timeout             90;
  proxy_send_timeout                90;
  proxy_read_timeout                90;
  proxy_buffers                     32 4k;
  proxy_buffer_size                 8k;
  proxy_set_header                  Host $host;
  proxy_set_header                  X-Real-IP $remote_addr;
  proxy_set_header                  X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_headers_hash_bucket_size    64;

  location ~ ^/static/  {
    include  /etc/nginx/mime.types;
    root /opt/web/powerdns-admin/app;

    location ~*  \.(jpg|jpeg|png|gif)$ {
      expires 365d;
    }

    location ~* ^.+.(css|js)$ {
      expires 7d;
    }
  }

  location / {
    proxy_pass            http://unix:/opt/web/powerdns-admin/powerdns-admin.sock;
    proxy_read_timeout    120;
    proxy_connect_timeout 120;
    proxy_redirect        off;
  }

}
```

Replacing `<YOUR DOMAIN NAME HERE>` with a domain name or subdomain pointing to your server that you'd like to access the panel on. This isn't strictly necessary, you'll be able to access your server at `http://<your server IP>` after this Nginx configuration is complete. But a domain is necessary to set up Let's Encrypt TLS, which is what we'll be doing in the next step.

If you don't have any other websites on this box, we're going to delete Nginx's built in default site now:

```
sudo rm /etc/nginx/sites-enabled/default
```

Now, restart Nginx to apply your changes:

```
sudo systemctl restart nginx
```

That should be it! PowerDNS-Admin should be accessible at `http://<your server IP or domain>`, next we'll setup TLS for additional in-transit security.

## Configuring Let's Encrypt

We're going to use Let's Encrypt's `certbot` program to issue certificates. Adding the certbot repo and installing the software is just a couple simple commands:

```
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

Now, we just issue this command using certbot and the nginx plugin to issue a certificate for your control panel:

```
sudo certbot --nginx
```

Since we added the domain in the configuration, it should pick it up automatically. Just follow through the email prompts and you should be good to go. If it issues a certificate successfully, it will ask you if you'd like to redirect all `http://` traffic to `https://`. There's really no reason not to do so, so I'd recommend choosing that option.

Renewals should be automated with this package, so no need to worry about certificate expiration, as long as your domain or subdomain stays pointed to your nameserver.

## Installing PowerDNS-Admin

Almost finished! There's just some final configuration required to get up and running. Browse to `https://your.domain.here` to access the web interface. There should be a link to register a new user, use that to register your administrator account.

After signing in, you'll be greeted with a page requesting your PowerDNS credentials (if you aren't, navigate to `/admin/setting/pdns`). For **PDNS API URL**, you'll enter `http://127.0.0.1:8081/`, because the PowerDNS server is hosted on the same server as this web interface.

For **PDNS API Key**, you'll want to enter the API key we created in the PowerDNS configuration in the previous install. This will give PowerDNS-Admin access to your PowerDNS backend.

PDNS Version should be pre-filled with the latest version of PowerDNS, which should be fine. At the time of writing, it's `4.1.1`.

Click Update and you should be done!

## Conclusion

After the completion of these guides, you should have both a PowerDNS server setup, complete with master and slave replication, as well as an administrative web panel for you and your users to manage zones on your servers. Changes made in the panel should be instantly updates across your entire installation.

I hope you found this guide helpful! If there's something you feel I missed, please feel free to reach out and let me know.
