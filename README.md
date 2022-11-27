
# Linux Server Configuration for Python Flask Application Hosted on Amazon (AWS) Lightsail

This guide will detail a server setup for a web application using the [Flask](https://flask.palletsprojects.com/en/2.2.x/) framework. An instance on Amazon's [Lightsail](https://aws.amazon.com/lightsail/) is used for hosting - providing a Linux server at a low, fixed cost. This guide covers configuration for:

| Software                                                           | Purpose                        |
| :----------------------------------------------------------------- | :----------------------------- |
| [Amazon (AWS) Lightsail instance](#amazon-aws-lightsail-instance)  | Host provider                  |
| Linux / Ubuntu                                                     | Operating system               |
| Nginx                                                              | Web server                     |
| Let's Encrypt                                                      | SSL certificate                |
| Supervisor                                                         | Manage Gunicorn processes      |
| Gunicorn                                                           | Python WSGI server             |
| Flask                                                              | Python web framework           |

## Amazon (AWS) Lightsail instance

1. Create an account or login to AWS Lightsail, https://lightsail.aws.amazon.com.
2. On the **Instances** tab, create an Ubuntu LTS instance
3. On the **Networking** Create a static IP address and attach it to your instance.
4. On the **Instances** tab, find the 'Manage' option for your instance and enable HTTPS (port 443) on the 'Networking' tab for both the IPv4 and IPv6 firewalls.

### SSH connection

From the **Instances** tab in the Lightsail web interface, you can start an SSH session from your browser using the "Connect" option. Alternatively, you can:

1. From the **Account** navigation menu in Lightsail, choose "Account" and then move to the "SSH keys" tab. You'll be able to download a default SSH key from this page.
2. Move the key file to the appropriate directory in your system. Depending on your system, you may need to set permissions for the key, e.g. `chmod 400 key.pem` 
3. SSH into the server, e.g. `ssh -i ~/.ssh/key.pem ubuntu@11.111.11.11`

## Initial updates

Run updates for installed packages:

```
sudo apt-get update
sudo apt-get upgrade
```

## Secure the server

1. Disallow root login and password logins:

```bash
# edit the configuration file
sudo nano /etc/ssh/sshd_config

# set these lines, if not already set
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes

# restart the service
sudo service ssh restart
```

## Install dependencies

1. Install python, supervisor, nginx, git

```bash
sudo apt-get -y update
sudo apt-get -y install python3 python3-venv python3-dev
sudo apt-get -y install supervisor nginx git
```

If a more recent version of Python is need, you can [install Python 3.11 and set it as the default](https://www.debugpoint.com/install-python-3-11-ubuntu/).

## Install the application

1. Checkout application files, create a virtual enviornment and install application dependencies

```bash
# example checkout
cd ~
git clone https://github.com/dandalpiaz/transit-cu.git
cd transit-cu

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## Set up Gunicorn and Supervisor

1. Create a supervisor configuration file.

```bash
sudo nano /etc/supervisor/conf.d/transit.conf

# example file
[program:transit-cu]
command=/home/ubuntu/transit-cu/venv/bin/gunicorn -b localhost:8000 -w 3 transit:app
directory=/home/ubuntu/transit-cu
user=ubuntu
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
```

2. Reload supervisor

```bash
sudo supervisorctl reload
sudo supervisorctl status
```

## Setup Nginx

1. Remove the default configuration file and create a new one.

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/transit

# example file
server {
    listen 80;
    listen [::]:80;

    server_name transitcu.com;

    location / {
        proxy_pass http://localhost:8000;
    }

    access_log /var/log/transit_access.log;
    error_log /var/log/transit_error.log;
}
```

2. Check the syntax of the configuration file and reload Nginx

```bash
sudo nginx -t
sudo service nginx reload
```

## Setup Let's Encrypt

1. Install the package

```bash
sudo add-apt-repository ppa:certbot/certbot
sudo apt install python3-certbot-nginx
```

2. Run certbot for your domain

```bash
sudo certbot --nginx -d transitcu.com
```

3. Do a dry-run to make sure setup is correct

```bash
sudo certbot renew --dry-run
```

## TODO

- add ufw configuration?
- add database configuration

## References

- [The Flask Mega-Tutorial Part XVII: Deployment on Linux](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)
