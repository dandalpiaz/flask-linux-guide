
# Flask in a Production Environment - FAQs

This guide will detail a server setup for a web application using the [Flask](https://flask.palletsprojects.com/en/2.2.x/) framework. An instance on Amazon's [Lightsail](https://aws.amazon.com/lightsail/) is used for hosting - providing a Linux server at a low, fixed cost. Don't use headings beyond h2 to avoid confusion?

## Table of Contents

- [Where can I **host** a Flask application?](#where-can-i-host-a-flask-application)
- [How can I use a **specific version of Python** for my app?](#how-can-i-use-a-specific-version-of-python-for-my-app)
- [How do I **protect** my app during **development**?](#how-do-i-protect-my-app-during-development)

## Where can I host a Flask application?

1. Create an account or login to AWS Lightsail, https://lightsail.aws.amazon.com.
2. On the **Instances** tab, create an Ubuntu 20.x LTS instance (OS Only)
3. On the **Networking** tab, create a static IP address and attach it to your instance.
4. On the **Instances** tab, find the 'Manage' option for your instance and enable HTTPS (port 443) on the 'Networking' tab for both the IPv4 and IPv6 firewalls.
5. If you will be using a custom domain, set up a blank "A" record with your DNS provider and point it at the static IP address for your instance.
6. SSH configuration:
    1. From the **Account** navigation menu in Lightsail, choose "Account" and then move to the "SSH keys" tab. You'll be able to download a default SSH key from this page.
    2. Move the key file to the appropriate directory in your system. Depending on your system, you may need to set permissions for the key, e.g. `chmod 400 key.pem` 
    3. SSH into the server, e.g. `ssh -i ~/.ssh/key.pem ubuntu@11.111.11.11`

## How can I use a specific version of Python for my app?

If you would like to use a more recent version of Python than what is availble in linux distribution, you can use the 'deadsnakes' PPA to add a newer version alongside the existing one. When creating the venv [[[tbd command]]]

```
sudo add-apt-repository ppa:deadsnakes/ppa

sudo apt update

sudo apt install python3.12

sudo apt-get install python3.12-venv
```

## How do I protect my app during development?

Create a username and password with Nginx:

```
sudo sh -c "echo -n 'myusername:' >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"
```

Edit your configuration file (e.g. `sudo nano /etc/nginx/sites-enabled/appname` ) and add 'auth_basic' lines:

```
...

location / {
    proxy_pass http://localhost:8000;
    auth_basic "Restricte Content";
    auth_basic_user_file /etc/nginx/.htpasswd;
}

...
```

Check the syntax of the configuration file, `sudo nginx -t` and reload Nginx `sudo service nginx reload` 

## Linux / Ubuntu

1. Install packages for python, let's encrypt, supervisor, nginx, git:
    - `sudo apt-get -y update`
    - `sudo apt-get -y install certbot python3-certbot-nginx`
    - `sudo apt-get -y install supervisor nginx git`
2. Run upgrades with `sudo apt-get upgrade`
    - You may have to do a couple rounds of updates and reboot in-between, `sudo reboot`
3. Disallow root login and password logins, `sudo nano /etc/ssh/sshd_config`
    ```
    # set these lines, if not already set
    PermitRootLogin no
    PubkeyAuthentication yes
    PasswordAuthentication no
    ChallengeResponseAuthentication no
    ```
4. Restart the SSH service, `sudo service ssh restart`

## Nginx

1. Remove the default configuration file, `sudo rm /etc/nginx/sites-enabled/default` and create a new one `sudo nano /etc/nginx/sites-enabled/appname`:
    ```
    server {
        listen 80;
        listen [::]:80;

        server_name example.com;

        location / {
            proxy_pass http://localhost:8000;
        }

        access_log /var/log/appname_access.log;
        error_log /var/log/appname_error.log;
        client_max_body_size 5M;
    }
    ```
2. Check the syntax of the configuration file, `sudo nginx -t` and reload Nginx `sudo service nginx reload`

## Let's Encrypt

1. If you haven't already, add a DNS record for your custom domain and point it to your Lightsail instance's static IP
2. Run certbot for your domain, `sudo certbot --nginx -d example.com`
3. Do a dry-run to make sure setup is correct, `sudo certbot renew --dry-run`

## Supervisor and Gunicorn

1. Create a supervisor configuration file, `sudo nano /etc/supervisor/conf.d/appname.conf`:
    ```
    [program:appdirectory]
    command=/home/ubuntu/appdirectory/venv/bin/gunicorn -b localhost:8000 -w 3 appname:app
    directory=/home/ubuntu/appdirectory
    user=ubuntu
    autostart=true
    autorestart=true
    stopasgroup=true
    killasgroup=true
    ```

## Flask

1. Configure an SSH key on your Lightsail instance for GitHub clones/pulls:
    - `cd ~/.ssh`
    - `ssh-keygen -t ed25519 -C "example@example.com"`
        - To avoid having to add the key to the ssh-agent, don't give the key a custom name (will default to "id_ed25519"), and (optionally) set a passphrase
    - Add the public key as a 'deploy key' to your GitHub repo, e.g. https://github.com/username/reponame/settings/keys
2. Checkout application files, create a virtual enviornment and install application dependencies:
    - `cd ~`
    - `git clone git@github.com:username/example-repo.git`
    - `cd example-repo`
    - `python3.11 -m venv venv`
    - `source venv/bin/activate`
    - `pip3 install -r requirements.txt`
    - Set configuration variables and setup the database, if necessary, e.g. `flask db upgrade`
3. Reload supervisor:
    - `sudo supervisorctl reload`
    - `sudo supervisorctl status`
    
To make udpates to the application, you can do a `git pull` on the repo and then run `sudo supervisorctl reload` .



