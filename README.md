
# Linux Server Configuration for Python Flask Application Hosted on Amazon (AWS) Lightsail

This guide will detail a server setup for a web application using the [Flask](https://flask.palletsprojects.com/en/2.2.x/) framework. An instance on Amazon's [Lightsail](https://aws.amazon.com/lightsail/) is used for hosting - providing a Linux server at a low, fixed cost. This guide covers configuration for:

![](logos.png)

| Software                                                           | Purpose                        |
| :----------------------------------------------------------------- | :----------------------------- |
| [Amazon (AWS) Lightsail instance](#amazon-aws-lightsail-instance)  | Host provider                  |
| [Linux / Ubuntu](#linux--ubuntu)                                   | Operating system               |
| [Nginx](#nginx)                                                    | Web server                     |
| [Let's Encrypt](#lets-encrypt)                                     | SSL certificate                |
| [Supervisor](#supervisor-and-gunicorn)                             | Manage Gunicorn processes      |
| [Gunicorn](#supervisor-and-gunicorn)                               | Python WSGI server             |
| [Flask](#flask)                                                    | Python web framework           |

## Amazon (AWS) Lightsail instance

1. Create an account or login to AWS Lightsail, https://lightsail.aws.amazon.com.
2. On the **Instances** tab, create an Ubuntu 20.x LTS instance (OS Only)
3. On the **Networking** tab, create a static IP address and attach it to your instance.
4. On the **Instances** tab, find the 'Manage' option for your instance and enable HTTPS (port 443) on the 'Networking' tab for both the IPv4 and IPv6 firewalls.

### SSH connection

From the **Instances** tab in the Lightsail web interface, you can start an SSH session from your browser using the "Connect" option. For extra convenience, you can bookmark the URL for the session for your instance (e.g. https://lightsail.aws.amazon.com/ls/remote/us-east-2/instances/instance-name/terminal?protocol=ssh). Alternatively, you can:

1. From the **Account** navigation menu in Lightsail, choose "Account" and then move to the "SSH keys" tab. You'll be able to download a default SSH key from this page.
2. Move the key file to the appropriate directory in your system. Depending on your system, you may need to set permissions for the key, e.g. `chmod 400 key.pem` 
3. SSH into the server, e.g. `ssh -i ~/.ssh/key.pem ubuntu@11.111.11.11`

## Python (WIP)

If you would like to use a more recent version of Python than what is availble in the Ubuntu LTS instance, you can use the 'deadsnakes' PPA to add a newer version alongside the existing one.

1. Install Python 3.11.x:
    ```
    sudo add-apt-repository ppa:deadsnakes/ppa

    sudo apt update

    sudo apt install python3.11
    ```

## Linux / Ubuntu

1. Install packages for python, let's encrypt, supervisor, nginx, git:
    - `sudo apt-get -y update`
    - `sudo apt-get -y install python3.11-venv certbot python3.11-certbot-nginx`
    - `sudo apt-get -y install supervisor nginx git`
2. Run upgrades with `sudo apt-get upgrade`
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

1. Optionally configure an SSH key on your Lightsail instance for GitHub clones/pulls:
    - `cd ~/.ssh`
    - `ssh-keygen -t ed25519 -C "example@example.com"`
        - To avoid having to add the key to the ssh-agent, don't give the key a custom name (will default to "id_ed25519") and don't set a passphrase
    - Add the public key to your GitHub profile at https://github.com/settings/keys
2. Checkout application files, create a virtual enviornment and install application dependencies:
    - `cd ~`
    - `git clone https://github.com/username/example-repo.git`
    - `cd example-repo`
    - `python3.11 -m venv venv`
    - `source venv/bin/activate`
    - `pip3 install -r requirements.txt`
    - Set configuration variables and setup the database, if necessary
3. Reload supervisor:
    - `sudo supervisorctl reload`
    - `sudo supervisorctl status`
    
To make udpates to the application, you can do a `git pull` on the repo and then run `sudo supervisorctl reload` .

## (optional) Password protect during development

1. Create a username and password with Nginx:
    - `sudo sh -c "echo -n 'myusername:' >> /etc/nginx/.htpasswd"`
    - `sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"`
2. Edit your configuration file and add 'auth_basic' lines:
    ```
    ...
    
    location / {
        proxy_pass http://localhost:8000;
        auth_basic "Restricte Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
    
    ...
    ```
3. Check the syntax of the configuration file, `sudo nginx -t` and reload Nginx `sudo service nginx reload` 

## TODO

- add ufw configuration?
- add database configuration, other flask configuration?

## References

- [The Flask Mega-Tutorial Part XVII: Deployment on Linux](https://blog.miguelgrinberg.com/post/the-flask-mega-tutorial-part-xvii-deployment-on-linux)
