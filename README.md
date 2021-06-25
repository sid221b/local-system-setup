## 🔥NGINX + 🔐HTTPS setup for local development environment!
> *Tested and working on **macOS Big Sur 11.4***

> Setting up local development environment so server will run on *https://local.housing.com*


First we are gonna install [**nginx**](https://www.nginx.com/) for which we need [**brew**](https://brew.sh/) package manager which can be installed by following same link. 

Once you have brew installed run
```bash 
brew install nginx
```

### SSL setup
>We’ll be using [OpenSSL](https://www.openssl.org/) to generate all of our certificates.

Follow  steps below.

1. #####  Root SSL certificate
Generate a RSA-2048 key and save it to a file `rootCA.key`.
```bash
openssl genrsa -des3 -out rootCA.key 2048
```

You can use the key you generated to create a new Root SSL certificate. Save it to a file named`rootCA.pem`. This certificate will have a validity of 1,024 days. Feel free to change it to any number of days you want. You’ll also be prompted for other optional information.
```bash
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
**Fill details as follows in terminal**
>Country Name=IN
State or Province Name=Random
Locality Name=Random
Organization=Housing
Organization Unit Name=Technology
Common Name=local.housing.com
Email Address=your.email@housing.com

2. #####  Trust the root SSL certificate

Open Keychain Access on your Mac and go to the Certificates category in your System keychain. Once there, import the `rootCA.pem` using File > Import Items. Double click the imported certificate and change the “When using this certificate:” dropdown to `Always Trust` in the Trust section.

3. #####  Domain SSL Certificate
The root SSL certificate can now be used to issue a certificate specifically for your local development environment located at `https://local.housing.com/`.

Create a new OpenSSL configuration file `server.csr.cnf` so you can import these settings when creating a certificate instead of entering them on the command line.

```bash
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=IN
ST=Random
L=Random
O=Housing
OU=Technology
emailAddress=your.email@housing.com
CN = local.housing.com
```

Now Create a `v3.ext` file in order to create a [X509 v3 certificate](https://en.wikipedia.org/wiki/X.509). Notice how we’re specifying `subjectAltName` here.

```bash
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = local.housing.com
```

Once you've created these file now Create a certificate key for `https://local.housing.com` using the configuration settings stored in `server.csr.cnf`. This key is stored in `server.key`.
```bash
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key -config <path_of server.csr.cnf>
```

A certificate signing request is issued via the root SSL certificate we created earlier to create a domain certificate for `localhost`. The output is a certificate file called `server.crt`.
```bash
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256 -extfile < path_of v3.ext>
```

4. ##### Configure system to serve localhost on alias

Now edit let your system know to serve local assets when `https://local.housing.com` is accessed from browser by editing `/etc/hosts` file
```bash
sudo vim /etc/hosts
```
Make changes in above file to look like this
```bash
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost local.housing.com local.edge.housing.com
255.255.255.255	broadcasthost
::1             localhost
```

5. ##### Configure and run NGINX server
Create a file **`nginx-node.conf`** and  assign location of `server.crt` and  `server.key`  that we created in [Step 3 ⏫](#domain-ssl-certificate) to `ssl_certificate` ,    `ssl_certificate_key`

> *Its best to move `server.crt` and  `server.key` into a different directory so it doesn't clash with any other configurations.*
```bash
events {}
http {
    upstream backend {
        server 127.0.0.1:4001;
    }
    upstream zeus {
        server 127.0.0.1:4000;
    }
    # server {
    #     server_name local.housing.com;
    #     rewrite ^(.*) https://local.housing.com$1 permanent;
    # }
    server {
        listen               443 ssl;
        # ssl                  on;
        ssl_certificate      /Users/path_of_/server.crt;
        ssl_certificate_key  /Users/path_of_/server.key;
        ssl_ciphers          HIGH:!aNULL:!MD5;
        server_name          local.housing.com;
        rewrite ^(.*)/edge/rent-pay(.*) https://housing.com/edge/pay-rent$2 permanent;
        location / {
            proxy_pass  http://backend;
        }
        location /api {
            proxy_pass  http://zeus;
        }
        location /gambit {
            proxy_pass https://housing.com;
        }
        location /fortuna {
            proxy_pass https://beta7.housing.com;
        }
        location /apollo {
            proxy_pass https://beta7.housing.com;
        }

    }
}
```

 6. ##### Run NGINX Server
Now all that left is to run the server with finger crossed🤞🏻
```bash
sudo nginx -c <path_of_nginx-node.conf>
```

and to stop server run command

```bash
sudo nginx -s stop
```
