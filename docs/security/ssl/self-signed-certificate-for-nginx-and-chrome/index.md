
### 0. Change the host file:

Add your local domain to your hosts file:

```sh
$ sudo nano /etc/hosts
```
Your hosts file looks like this:

```
127.0.0.1       localhost
127.0.1.1       case-G41MT-S2P
127.0.0.1       mywebapp.dev www.mywebapp.dev
```

###1. Install and configure Nginx

```sh
$ cd
$ wget https://nginx.org/keys/nginx_signing.key

$ echo "# Nginx 2" | sudo tee -a /etc/apt/sources.list
$ echo "deb http://nginx.org/packages/ubuntu/ xenial nginx" | sudo tee -a /etc/apt/sources.list
$ echo "deb-src http://nginx.org/packages/ubuntu/ xenial nginx" | sudo tee -a /etc/apt/sources.list

$ sudo apt-get update && sudo apt-get install nginx -y
```

### 2. Configure Nginx web server

```sh
$ cd
$ sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/mywebapp.dev
$ sudo ln -s /etc/nginx/sites-available/mywebapp.dev /etc/nginx/sites-enabled/
$ sudo nano /etc/nginx/sites-available/mywebapp.dev
```

Let make some adjustment. Change those lines below:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
```

To

```
server {
    listen 80;
    listen [::]:80;
    server_name bitcommerce.dev www.bitcommerce.dev;

	return 302 https://$server_name$request_uri;

	root /var/www/html/mywebappdev;
```

Now add the block below after your server block:

```
server {

    # SSL configuration

    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name mywebapp.dev www.mywebapp.dev;

    root /home/david/dev/js/mywebappdev;
    index index.html index.htm;

    ssl on;
    ssl_certificate /etc/nginx/ssl/mywebapp.dev.crt;
    ssl_certificate_key /etc/nginx/ssl/mywebapp.dev.key;

    ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

    location / {
            try_files $uri $uri/ =404;
    }
}
```

It's a second server block. Save and Exit.

Create a root folder for your web files:

```sh
$ sudo mkdir /var/www/html/mywebapp
```

### 3. Create a root CA cert

First, let's create a folder for our root certificate.

```sh
$ mkdir mycert
$ cd mycert
```

After create a folder for our root certificate...

```sh
$ openssl genrsa -des3 -out rootCA.key 2048
```

You'll be asked for:

**Enter pass phrase for rootCA.key**

Repeat the passphrase key:

**Verifying - Enter pass phrase for rootCA.key**

Now generate the root pem:

```sh
$ openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```

You'll be asked to complete some info about your local domain

**Enter pass phrase for rootCA.key**
**Country Name (2 letter code) [AU]:**
**State or Province Name (full name) [Some-State]:**
**Locality Name (eg, city) []:**
**Organization Name (eg, company) [Internet Widgits Pty Ltd]:**
**Organizational Unit Name (eg, section) []:**

Pay attension to answer the next question, put ***localhost*** for local server for exemple.

**Common Name (e.g. server FQDN or YOUR name) []:** 
**Email Address []:**

### 4. Create private key for your local domain

Use the rootCA to create private key for your local domain.

In my case: **mywebapp.dev**

```sh
$ sudo mkdir /etc/nginx/ssl
$ sudo openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout /etc/nginx/ssl/mywebapp.dev.key
$ sudo nano v3.ext
```

Paste this peace inside of your v3.ext file:

```authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = www.mywebapp.dev
```

Change the DNS to your local domain name.

Save and close the v3.ext file.

Now create the cert for your local domain with:

```sh
$ sudo openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out /etc/nginx/ssl/mywebapp.dev.crt -days 1000 -sha256 -extfile v3.ext
```

Your local domain cert will be stored inside ***/etc/nginx/ssl/***

Now test the nginx configuration file:

```sh
$ sudo nginx -t
```

If everything is OK, restart nginx:

```sh
$ sudo service nginx restart
```
For Google Chrome:

Access `chrome://settings/certificates` in Chrome or Settings -> Advanced -> Manage certificates -> AUTHRORITIES.

Click `IMPORT`, and select  `mycert/rootCA.pem` created earlier.

When promoted, select  `Trust this certificate for identifying websites` and click `OK`.

Reference:
https://code.luasoftware.com/tutorials/nginx/self-signed-ssl-for-nginx-and-chrome/

