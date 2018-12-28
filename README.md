
# Linux Server Configuration for Flask Application on Amazon Lightsail

## Amazon Lightsail

1. Create an Ubuntu 18 LTS instance in Lightsail.
2. Create a static IP address and attach it to your instance.
3. Enable HTTPS (port 443) on Lightsail firewall.

## SSH configuration

1. Download your SSH key from Lightsail.
2. If you are using Windows Subsystem for Linux:
    ```bash
    # navigate to the directory that contains the key
    cd /mnt/d/ssh

    # copy the key to the ssh folder in the subsystem
    cp LightSailDefaultPrivateKey-us-east-2.pem ~/.ssh/key.pem

    # set permissions on the key
    chmod 400 key.pem
    ```
3. SSH into the server, e.g:
    ```bash
    # example
    ssh -i ~/.ssh/key.pem ubuntu@11.111.11.11
    ```

## Secure the server

1. Disallow root login and password logins:
    ```bash
    # edit the configuration file
    sudo nano /etc/ssh/sshd_config

    # set these two lines
    PermitRootLogin no
    PasswordAuthentication no

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
    command=/home/ubuntu/transit-cu/venv/bin/gunicorn -b localhost:8000 -w 4 transit:app
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
    sudo apt install python-certbot-nginx
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

- go back and set up access and error logs
- add ufw configuration?
- add database configuration

## References

- [The Flask Mega-Tutorial Part XVII: Deployment on Linux](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)