
# Linux Server Configuration for a Python Flask Application on a Debian-based Host

![](logos.png)

This guide details the setup of a web application using the [Flask](https://flask.palletsprojects.com/en/2.2.x/) framework on a 'traditional' Debian-based Linux server (Debian, Ubuntu, etc.). Configuration for specific software/packages is included, but can be swapped out as needed. The guide will make use of "appname.com" and variations on that name for different purposes:

- `appname.com` - the site's domain
- `/home/admin/appname.com` - the application's directory on the server
- `/etc/nginx/sites-enabled/appname.conf` - the web server configuration file
- `/etc/supervisor/conf.d/appname.conf` - the supervisor configuration file
- `appname` - the name of the database, and the database user
- `appname.py` - the name of the main/start file for the application

## Table of Contents

- Server Setup
    - [Debian Instance](#debian-instance)
    - [Install Python](#install-python)
    - [Install Linux Packages](#install-linux-packages)
    - [SSH Securing](#ssh-securing)
    - [Nginx](#nginx)
    - [Let's Encrypt](#lets-encrypt)
    - [Hosts File](#hosts-file)
    - [Supervisor and Gunicorn](#supervisor-and-gunicorn)
    - [Flask](#flask)
    - [MariaDB](#mariadb)
    - [Swap File](#swap-file)
    - [Backups](#backups)
    - [Restoring](#restoring)
    - [Protect DEV](#protect-dev)
    - [References](#references)
- Other Services, Extensions, etc.
    - [Object Storage](#object-storage)
    - [SMTP Email](#smtp-email)
    - [Cloudflare CDN](#cloudflare-cdn)
    - [Cloudflare Turnstile](#cloudflare-turnstile)
- Maintenance
    - [Python and Packages](#python-and-packages)
    - [Debian Core and Packages](#debian-core-and-packages)
    - [Deploying Changes](#deploying-changes)
    - [Command Line](#command-line)

## Server Setup

### Debian Instance

Using the Amazon Web Services 'Lightsail' service as an example host (other hosts that offer a similar option for a Debian-based VPS include [DigitalOcean](https://www.digitalocean.com/pricing/droplets) and [Linode](https://www.linode.com/pricing/#compute-shared)). Assumes that you have a domain set up through a registrar and DNS provider that can be pointed at the instance's IP address.

1. Create an account or login to AWS Lightsail, https://lightsail.aws.amazon.com.
2. On the **Instances** tab, create an Debian 12.x LTS instance (OS Only)
    - If you'll be hosting a database on the server, you'll probably want at least 1-2 GB RAM for the instance
    - Note, you'll likely want an instance with an IPv4 and IPv6 address unless your application can function with only IPv6
3. On the **Networking** tab, create a static IP address and attach it to your instance.
4. On the **Instances** tab, find the 'Manage' option for your instance and enable HTTPS (port 443) on the 'Networking' tab for both the IPv4 and IPv6 firewalls.
5. If you will be using a custom domain, set up the DNS record(s) with your DNS provider and point it at the static IP address for your instance.
6. Set up an SSH connection:
    1. From the **Account** navigation menu in Lightsail, choose "Account" and then move to the "SSH keys" tab. You'll be able to download a default SSH key from this page.
    2. Move the key file to the appropriate directory in your system. Depending on your system, you may need to set permissions for the key, e.g. `chmod 400 key.pem` 
    3. SSH into the server, e.g. `ssh -i ~/.ssh/key.pem admin@11.111.11.11`

### Install Python

If you would like to use a more recent version of Python than what is availble in the Linux distribution, you can use the 'deadsnakes' PPA to add a newer version alongside the existing one:

```
sudo add-apt-repository ppa:deadsnakes/ppa

sudo apt update

sudo apt install python3.12

sudo apt install python3.12-venv
```

Otherwise, if you're happy with distribution's version, you'll just need to install the `venv` package that matches the current python version:

```
sudo apt update

sudo apt install python3.11-venv
```

### Install Linux Packages

Install packages commonly used with Flask applications. Nginx is used for the web server (note that you'll want to [Tell Flask it is Behind a Proxy](https://flask.palletsprojects.com/en/3.0.x/deploying/proxy_fix/)), Supervisor manages Flask processes/workers, MariaDB is used for the database, Memcached is used for in-memory storage (useful for extensions like [Flask-Limiter](https://flask-limiter.readthedocs.io/en/stable/)).

```
sudo apt update

sudo apt install nginx supervisor mariadb-server memcached

sudo apt install certbot python3-certbot-nginx

sudo apt install git htop cron
```

At this point you may want to do a round or two of updates:

```
sudo apt update

sudo apt upgrade

sudo reboot
```

### SSH Securing

Disallow root login and password logins, `sudo nano /etc/ssh/sshd_config`:

```
# set these lines, if not already set
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
```

And restart the SSH service, sudo service ssh restart .

### Nginx

Remove the default configuration file, `sudo rm /etc/nginx/sites-enabled/default` and create a new one `sudo nano /etc/nginx/sites-enabled/appname.conf`:

```
server {
    server_name www.appname.com;
    return 301 $scheme://appname.com$request_uri;
}

server {
    listen 80;
    server_name appname.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Prefix /;
    }

    location /static {
        alias /home/admin/appname.com/app/static;
        expires 30d;
    }

	location /robots.txt {
        alias /home/admin/appname.com/app/static/robots.txt;
        expires 30d;
    }

    access_log /var/log/appname_access.log;
    error_log /var/log/appname_error.log;
    client_max_body_size 5M;
}
```

These rules assume that you have a 'www' record that you want to redirect to a non-www record. It adds configuration specific to Nginx per the [flask nginx](https://flask.palletsprojects.com/en/2.3.x/deploying/nginx/) guide. It assumes that your app is hosted in a directory like `/home/admin/appname.com`, including a `static` directory for files.

Check the syntax of the configuration file, `sudo nginx -t` and reload Nginx `sudo service nginx reload`.

### Let's Encrypt

Run certbot for your domain, `sudo certbot --nginx -d appname.com -d www.appname.com` and if desired, do a dry-run to make sure setup is correct, `sudo certbot renew --dry-run`. The certbot will make modifications to the `appname` Nginx configuration file.

### Hosts File

In Debian, it seems to be necessary to remove 'localhost' from the IPv6 line in the `/etc/hosts` file in order for the Flask-Limiter to work as expected, leaving:

```
127.0.0.1       localhost
::1             ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

### Supervisor and Gunicorn

As an example, using the [Gunicorn](https://gunicorn.org/) python package for a WSGI HTTP server. Create a supervisor configuration file, `sudo nano /etc/supervisor/conf.d/appname.conf`:

```
[program:appname.com]
command=/home/admin/appname.com/venv/bin/gunicorn -b localhost:8000 -w 4 appname:app
directory=/home/admin/appname.com
user=admin
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
```

This will create 4 'workers' for the Flask application to use to handle requests.

### Flask

Create an SSH key on the instance for GitHub clones/pulls by doing `cd ~/.ssh` and then `ssh-keygen -t ed25519 -C "example@gmail.com"`. Don't give the key a custom name or passphrase to avoid having to add to ssh-agent each time. Then, add the public key as a 'deploy key' to the GitHub repo, e.g. https://github.com/username/appname.com/settings/keys .

Now, checkout the application create a virtual environment and install the application dependencies:

```
cd ~

git clone git@github.com:username/appname.com.git

cd appname.com

python3.11 -m venv venv

source venv/bin/activate

pip3 install -r requirements.txt
```

If you have environment variables stored in `.flaskenv` and `.env` files, add those files in (they shouldn't be stored in git/GitHub).

### MariaDB

Setup the MariaDB database:

```
sudo mysql_secure_installation
# don't need root user password (uses socket auth instead)
# but do clean up default tables/users via the prompts

sudo mysql -u root

CREATE DATABASE appname;
CREATE USER 'appname'@'localhost' IDENTIFIED BY 'password-here';
GRANT ALL PRIVILEGES ON appname.* TO 'appname'@'localhost';
FLUSH PRIVILEGES;

exit;
```

Add a configuration variable for the database to the `.env` file, for example, if you're using the [pymysql](https://pymysql.readthedocs.io/en/latest/) package/library:

```
DATABASE_URL=mysql+pymysql://appname:password-here@localhost:3306/appname
```

Reboot and initialize the database and reload. Note, the  `flask db upgrade` command comes from [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/):

```
sudo reboot

cd appname.com

source venv/bin/activate

flask db upgrade

sudo supervisorctl reload
```

### Swap File

Create a `/swapfile` in case of memory issues:

```
sudo fallocate -l 2G /swapfile

sudo chmod 600 /swapfile

sudo mkswap /swapfile

sudo swapon /swapfile

echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

sudo swapon --show
```

And adjust the 'swappiness':

```
sudo nano /etc/sysctl.d/99-sysctl.conf

vm.swappiness=10

sudo sysctl --system

sudo reboot
```

### Backups

An automatic backup of the database and files uploaded to the application can be configured using [rclone](https://rclone.org/downloads/). As an example, you could follow the [rclone dropbox](https://rclone.org/dropbox/) configuration instructions. Since the server doesn't have a web browser to complete the setup, this will require installing rclone on a different computer to get a token. For example, on Windows, download and extract rclone.exe and run with cmd:

```
rclone.exe authorize "dropbox"
```

Then, on the server, you can add a cron job for various backups and use rclone commands to send it to a Dropbox folder in your account. Run `crontab -e` and add to the file:

```
30 1 * * * cd /home/admin && sudo mysqldump appname > appname.sql
40 1 * * * tar -czvf /home/admin/uploads.tar.gz -C /home/admin/appname.com/app/uploads .
30 2 * * * rclone copy /home/admin/appname.sql dropbox:appname.com/backup-$(date +\%Y-\%m-\%d)
40 2 * * * rclone copy /home/admin/uploads.tar.gz dropbox:appname.com/backup-$(date +\%Y-\%m-\%d)
```

### Restoring

If a database restore is needed, you can remove the current database, download the backup SQL file from Dropbox (as an example), upload it to the server, and create a new database and import:

```
# locally:
scp -i /c/ssh/key.pem appname.sql admin@11.111.11.11:/home/admin

# connect to server, then:
sudo mysql -u root

DROP DATABASE appname;
CREATE DATABASE appname;
exit;

sudo mysql -u root appname < appname.sql
```

To restore uploaded files, download the backup archive from Dropbox (as an example), upload it to the server, and copy:

```
# locally:
scp -i /c/ssh/key.pem uploads.tar.gz admin@11.111.11.11:/home/admin

# connect to server, then:
tar -xzf uploads.tar.gz -C /home/admin/appname.com/app/uploads
```

### Protect DEV

Optionally, while your app is still under development, you can put a basic password on the site. Create a username and password with Nginx, `sudo sh -c "echo -n 'exampleusername:' >> /etc/nginx/.htpasswd"` and `sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"` . Then edit the confituration file `sudo nano /etc/nginx/sites-enabled/appname.conf`:

```
...

location / {
    proxy_pass http://localhost:8000;
	...
	...
    auth_basic "Restricted Content";
    auth_basic_user_file /etc/nginx/.htpasswd;
}

...
```

And then check the syntax, `sudo nginx -t` and reload `sudo service nginx reload`.

### References

Miguel Grinberg's [Flask Mega Tutorial](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux) was referenced for parts of these server configurations.

## Other Services, Extensions, etc.

### Object Storage

Web applications will often need storage for uploaded files beyond the available storage space on the server, for both capacity and performance reasons. Different 'object storage' services are available. Using a [Lightsail bucket](https://lightsail.aws.amazon.com/ls/webapp/home/storage) as a simple example:

1. Login to Lightsail, switch to the 'Storage' tab and click the 'Create bucket' button
2. Set bucket location to to your desired region, choose storage amount
3. Manage the bucket and switch to the 'Permission' tab
4. Create an 'Access key' and copy the 'access key id', 'secret key', 'bucket name' and 'region' to the `.env` file

To work with the bucket, you'll likely want the [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html) package installed for your Flask app.

### SMTP Email

Web applications needing to send email (e.g. for account management) often use an SMTP service. Using [Amazon Simple Email Service (SES)](https://aws.amazon.com/console/) as an example, you would set this up by:

1. Creating the resource in your desired region
2. From 'Verified identites', complete DKIM verfication using DNS records for appname.com
3. Also verify a 'Custom MAIL FROM domain' for mail.appname.com
4. Create an IAM user which can then create an SMTP key
5. Copy the 'server address', 'username' and 'SES key/password' to your `.env` file for use in the application's configuration
6. Request 'production access' from Amazon (requires a brief justification for your intended usage)

To send mail from the application, you'll likely want something like the [Flask-Mail](https://pypi.org/project/Flask-Mail/) extension for your app.

### Cloudflare CDN

For better performance globally, applications will often make use of a Content Delivery Network (CDN). [Cloudflare](https://www.cloudflare.com/) offers a free CDN option. The setup will depend on how your DNS is configured, but steps might include:

1. Importing/adding DNS records into Cloudflare from your current DNS provider
2. Switching your 'name servers' to the values provided in the CLoudflare dashboard
3. Setting options like 'Always Use HTTPS', 'Automatic HTTPS Rewrites' and 'Full (strict) mode'
4. Enabling DNSSEC, which will require adding a DS record to the DNS
5. Following Cloudflare suggestions for DNS records related to email spoofing: SPF, DKIM, DMARC, etc. 

### Cloudflare Turnstile

Web applications will often use a CAPTCHA-type challenge to verify that users are human and not bots. These are commonly added to important forms in the application, like sign up forms. Using [Cloudflare Turnstile](https://www.cloudflare.com/) as an example:

1. Select 'Turnstile' from the sidebar and click the 'Add site' button
2. Create a 'Managed' widget and copy the 'Site Key' and 'Secret key' to your `.env` file
3. Add 'appname.com' as an allowed domain
    - Adding 'localhost' can be useful for testing, but consider removing 'localhost' from the allowed domains when moving to production and use the [test keys](https://developers.cloudflare.com/turnstile/reference/testing/) instead

You can use the [Flask-Turnstile](https://pypi.org/project/Flask-Turnstile/) extension for Cloudflare CAPTCHAs, or something like [Flask-reCaptcha](https://pypi.org/project/Flask-reCaptcha/) if you're using Google CAPTCHAs.

## Maintenance

### Python and Packages

On Debian, Python core updates should be available through the default package, or deadsnakes. For Python packages, make sure that Dependabot alerts are enabled in your GitHub repo. When a vulnerability is found, you can specify the version to upgrade to in the `requirements.txt` file and then follow 'Deploying Changes' below (after testing).

### Debian Core and Packages

Run updates with:

```
sudo apt update

sudo apt upgrade

sudo reboot
```

### Deploying Changes

```
cd ~/appname.com

git pull

# start virtual environment (if needed)
source venv/bin/activate

# package updates (if needed)
pip3 install -r requirements.txt

# database updates (if needed)
flask db upgrade

sudo supervisorctl reload
```

## Command Line

In your main `appname.py` file, if you can define parts of your database model that you can interact with from a command line session:

```
from app import app, db
from app.models import User, Page

@app.shell_context_processor
def make_shell_context():
    return {'db': db, 'User': User, 'Page': Page}
```

You can then start a command line session with:

```
source venv/bin/activate

flask shell
```

And query data or make modifications like:

```
users = User.query.all()

for u in users:
    print(u.id, u.email)

user_to_delete = User.query.get(2)

db.session.delete(u)

db.session.commit()
```
