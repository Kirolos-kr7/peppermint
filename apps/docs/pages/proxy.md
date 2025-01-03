## Peppermint behind a reverse proxy

Sometimes, individuals & organisations like to put applications behind a reverse proxy, the main reason for this is to allow applications to be accessed on specific domains.

An example of this would be a helpdesk company running peppermint having it accessible to employee's through their domain name.

```
https://support.example.com
```

Behind the scenes you'll most likely use nginx, trafik or haproxy to achieve this goal. In this example we will be using nginx on debian.

In the end, you should have a setup like so:

```
https://peppermint.example.com -> https://peppermintapi.example.com -> nginx -> peppermint docker
```

I know two different subdomains may seem a bit overkill, but this is the only way to achieve this goal.

## Setting up the box & nginx

In this example, I'm going to be using a node provided by linode pre-installed with docker, but any vm should be able to achieve this provided they have a static i.p address.
I wont be going into any detail regarding secuirty or best practices for setting up a linux machine, or we'll be here all day :)
This is going to be a straight forward, down to business guide.

With docker pre installed, we need to get the second piece of the puzzle installed, which is nginx.

Since this our first interaction which the machine, we'll need to update the package manager to get the latest listings:

```
sudo apt update
```

After this we can installed nginx

```
sudo apt install nginx
```

Before testing Nginx, the firewall software needs to be adjusted to allow access to the service.

List the application configurations that ufw knows how to work with by typing:

```
sudo ufw app list
```

This should show the list of application profiles

```
Output
Available applications:
...
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
...
```

As you can see, there are three profiles available for Nginx:

- Nginx Full: This profile opens both port 80 (normal, unencrypted web traffic) and port 443 (TLS/SSL encrypted traffic)
- Nginx HTTP: This profile opens only port 80 (normal, unencrypted web traffic)
- Nginx HTTPS: This profile opens only port 443 (TLS/SSL encrypted traffic)

It is recommended that you enable the most restrictive profile that will still allow the traffic you’ve configured. Since we haven’t configured SSL for our server yet in this guide, we will only need to allow traffic for HTTP on port 80.

Lets enable that now

```
sudo ufw allow 'Nginx HTTP'
```

Use https if you have SSL cert enabled.

### Checking nginx

To make sure nginx is running okay when can run the command:

```
systemctl status nginx
```

Which should display

```
Output
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-07-03 12:52:54 UTC; 4min 23s ago
     Docs: man:nginx(8)
 Main PID: 3942 (nginx)
    Tasks: 3 (limit: 4719)
   Memory: 6.1M
   CGroup: /system.slice/nginx.service
           ├─3942 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           ├─3943 nginx: worker process
           └─3944 nginx: worker process
```

Another way you can test this is working is by going to:

```
http://your_server_ip
```

And if everything is good you should see the welcome to nginx sign.

## Setting up peppermint

Now nginx is out the way and configured, we can now simply download the docker compose file from github:

```
wget https://raw.githubusercontent.com/Peppermint-Lab/Peppermint/master/docker-compose.yml
```

After this we should be able to run:

```
nano docker-compose.yml
```

and we'll be presented with an output like so:

```
version: "3.1"

services:
  peppermint_postgres:
    container_name: peppermint_postgres
    image: postgres:latest
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: peppermint
      POSTGRES_PASSWORD: 1234
      POSTGRES_DB: peppermint

  peppermint:
    container_name: peppermint
    image: pepperlabs/peppermint:latest
    ports:
      - 3000:3000
      - 5003:5003
    restart: always
    depends_on:
      - peppermint_postgres
    environment:
      DB_USERNAME: "peppermint"
      DB_PASSWORD: "1234"
      DB_HOST: "peppermint_postgres"
      SECRET: 'peppermint4life'

volumes:
 pgdata:
```

You could also reverse proxy the backend as well if you really wanted too, but for this example we'll just be doing the frontend.

Now for your base url: this is where your subdomain url is going to, if youre using nginx https with SSL, then make sure you change it accordingly.

After you enter the correct base url hit `ctrl + x` and then `y` to save. When this is complete run the command:

```
docker-compose up -d
```

This will pull the peppermint image & postgres and start the process of both containers. Once up, we have one finally more nginx config to take care of.

### Nginx config

Now that nginx is set up & both are containers are working, we can now implement the config file which is going to route our proxy to our subdomains.

I like to save this in the conf.d folder of nginx, it works for me and i never run into issues.

Start off by running:

```
nano /etc/nginx/conf.d
```

This will bring an editor up, in which you will paste

### Client Proxy Config

```
server {
    listen 80;
    listen [::]:80;
    server_name peppermint.example.com;
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_redirect off;
        proxy_read_timeout 5m;
    }
    client_max_body_size 10M;
}
```

Replace the server name with your url of choice, including subdomain and procced to save the file as

```
peppermint-client.conf
```

### API Proxy Config

```
server {
    listen 80;
    listen [::]:80;
    server_name peppermintapi.example.com;
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains" always;

    location / {
        proxy_pass http://127.0.0.1:5003;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_redirect off;
        proxy_read_timeout 5m;
    }
    client_max_body_size 10M;
}
```

Replace the server name with your url of choice, including subdomain and procced to save the file as, make sure this matches what you put in the docker-compose.yml file

```
peppermint-api.conf
```

### Restarting nginx

The final touch to the tale is you need to restart nginx for this to take full effect.

This can be achieved by running:

```
systemctl restart nginx
```

You should now be able to see peppermint running on your choosen subdomain.

I hope you found this guide usual :)

## Enabling SSL

To enable SSL, you will need to open some additional ports. You can do that by using the below command.

```
sudo ufw allow 'Nginx HTTPS'
```

We will be using Certbot to do the heavy listing. We are going to assume that this is a fresh new install of an OS of your choice.

### Installing Certbot

The below commands will install Python3, the Python VENV module, and the Augeas for the Apache plugin.

For APT based systems (Debian, Ubuntu)
```
sudo apt update
sudo apt install python3 python3-venv libaugeas0
```

For RPM based systems (Fedora, CentOS)
```
sudo dnf install python3 augeas-libs
```

We now need to set up a Python virtual environment using these commands
```
sudo python3 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
```

Once that is done, we can install Certbot using this command
```
sudo /opt/certbot/bin/pip install certbot certbot-nginx
```

Next, we need to run this command to ensure that the certbot command can be run
```
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
```

Congrats! Certbot is now installed! Now we can automatically enable SSL by using the below command, and following all of the prompts!
```
sudo certbot --nginx
```

Certificates issued by Certbot do expire, so you can opt to either manually renew, or you can use cron.
```
echo "0 0,12 * * * root /opt/certbot/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && sudo certbot renew -q" | sudo tee -a /etc/crontab > /dev/null
```

You should now be able to visit your site using https!

## Issues i discovered

An issue i discovered was that the api was not able to serve requests if it wasnt HTTPS, just bear that in mind.
