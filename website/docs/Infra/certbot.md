# Certbot

## Installing Certbot

Most universal way is to install Certbot via `pip`:

```shell
sudo apt install python3 python3-venv libaugeas0
sudo python3 -m venv /opt/certbot
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot

sudo certbot --nginx # for nginx
sudo certbot --apache # for apache/httpd, use pip install certbot-apache
```
