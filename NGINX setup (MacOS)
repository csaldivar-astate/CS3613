# NGINX setup (MacOS)

```
brew install nginx
```

You need to set up a self-signed certificate for HTTPS on localhost. Run these two lines.
```bash
mkdir -p /usr/local/etc/private
mkdir -p /usr/local/etc/certs
```

This is where your cert and key will go (just an organizational thing). Then run this line:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /usr/local/etc/private/nginx-selfsigned.key -out /usr/local/etc/certs/nginx-selfsigned.crt
```

You'll need to trust the cert
```
security import /usr/local/etc/certs/nginx-selfsigned.crt -k ~/Library/Keychains/login.keychain
```

Then open keychain access and go to your `login` keychain. Click `all items` in **Category** and scroll until you see `juniper.beer`. Double click and click **Trust** this will open a fold. You'll see several select boxes choose the one labeled: **When using this certificate:**  and choose the option `Always Trust`. The exit and you'll be prompted for your password.

To edit the nginx config open it with this line:
```bash
code /usr/local/etc/nginx/nginx.conf
```
Delete everything and replace it with

```bash
worker_processes  1;

events {
    worker_connections 1024;
}


http {    
    server {

            root /var/www/juniper.beer/html;
            index index.html index.htm index.nginx-debian.html;

            server_name juniper.beer www.juniper.beer;

            location / {
                    proxy_pass http://juniper.beer:9090;
                    proxy_http_version 1.1;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection 'upgrade';
                    proxy_set_header Host $host;
                    proxy_cache_bypass $http_upgrade;
            }

        listen [::]:443 ssl ipv6only=on;
        listen 443 ssl;
            ssl_certificate /usr/local/etc/certs/nginx-selfsigned.crt;
            ssl_certificate_key /usr/local/etc/private/nginx-selfsigned.key;


    }
    server {
        if ($host = www.juniper.beer) {
            return 301 https://$host$request_uri;
        }


        if ($host = juniper.beer) {
            return 301 https://$host$request_uri;
        }


            listen 80;
            listen [::]:80;

            server_name juniper.beer www.juniper.beer;
        return 404;
    }
}
```

Then finally restart nginx with:

```bash
sudo brew services restart nginx
```
