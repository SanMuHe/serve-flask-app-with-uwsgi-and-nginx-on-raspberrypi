# Serve Flask Applications with uWSGI and Nginx on RaspberryPi

## Table of Contents

- [Install Prerequisites](#install-prerequisites)
- [Configure uWSGI](#configure-uwsgi)
- [Configure Nginx](#configure-nginx)
- [Troubleshooting Tips](#troubleshooting-tips)
- [Reference](#reference)
- [License](#license)

## Install Prerequisites

Update your local package index and then install the packages with:

```bash
sudo apt-get update
sudo apt-get install python-pip python-dev nginx uwsgi uwsgi-plugin-python
```

## Configure uWSGI

### Create a uWSGI configuration file

Create a uWSGI configuration file

```bash
sudo nano /etc/uwsgi/vassals/myproject.ini
```

with the below content

```text
[uwsgi]
chdir = <folder to your flask application file>
plugins = python
socket = 127.0.0.1:4242
module = <your flask application name>
callable = app
process = 3
logger = file:/myproject/uwsgi.log
```

### Create an Upstart Script

Creating an Upstart script will allow the init system to automatically start uWSGI and serve our Flask application whenever the server boots.

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/emperor.uwsgi.service
```

with the below content

```text
[Unit]
Description=uWSGI Emperor
After=syslog.target

[Service]
ExecStart=/usr/bin/uwsgi --ini /etc/uwsgi/emperor.ini
# Requires systemd version 211 or newer
RuntimeDirectory=uwsgi
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

Create an `Emperor` configuration file:

```bash
sudo nano /etc/uwsgi/emperor.ini
```

with the below content

```text
[uwsgi]
emperor = /etc/uwsgi/vassals
```

Then you can start the service with:

```bash
systemctl start emperor.uwsgi.service
```

and check its status:

```bash
systemctl status emperor.uwsgi.service
```

You can stop the service with

```bash
systemctl stop emperor.uwsgi.service
```

## Configure Nginx

Create a new server block configuration in Nginx's `sites-available` directory:

```bash
sudo nano /etc/nginx/sites-available/myproject
```

with the below content

```text
server {
    listen      80;
    server_name <your domain name>;
    rewrite        ^ https://$server_name$request_uri? permanent;
}

server {
    listen      443;

    ssl on;
    ssl_certificate    <path to your certificate file>;
    ssl_certificate_key    <path to your certificate key file>;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_dhparam /etc/ssl/dhparams.pem;

    server_name <your domain name>;
    charset     utf-8;

    location / {
        uwsgi_pass 127.0.0.1:4242;
        include uwsgi_params;
    }
}
```

To enable the Nginx server block configuration we've just created, link the file to the sites-enabled directory:

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

With the file in that directory, we can test for syntax errors by typing:

```bash
sudo nginx -t
```

If this returns without indicating any issues, we can restart the Nginx process to read the our new config:

```bash
sudo service nginx restart
```

**Note:** For Nginx it is required to have all the certificates (one for your domain name and CA ones) combined in a single file. The certificate for your domain should be listed first in the file, followed by the chain of CA certificates.
If you have downloaded a complete CABundle file for your certificate, use the following command to combine all the certificates into a single file:

```bash
cat *yourdomainname*.crt *yourdomainname*.ca-bundle >> cert_chain.crt
```

## Troubleshooting Tips

### 502 Bad Gateway

Sometimes, after restart your RaspberryPi, you will get `502 Bad Gateway` error from nginx server.
One possible reason of this error is the uWSGI service is not correctly started.
Try to run `systemctl start emperor.uwsgi.service` again to start uWSGI service can fix this issue.

## Reference

- [How To Serve Flask Applications with uWSGI and Nginx on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-14-04)
- [Systemd](https://uwsgi-docs.readthedocs.io/en/latest/Systemd.html)

## License

&copy; 2018 SanMuHe

This repository is licensed under the MIT license. See `LICENSE` for details.