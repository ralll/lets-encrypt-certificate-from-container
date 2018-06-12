# Steps to create Let's Encrypt certificates with containers.

For developers with difficult to obtain the Let's Encrypt certificate due to variety of problems such as complex configurations and applications problems, It's possible to download the certificate to a docker volume and, then, install in the server.

## **Warning:**

> Be carefully with the certificates.

> **Don't use non-secure connections** with server and **review the firewall** and other **security measures**.

## Requirements

* DNS service configured. The site already runs in HTTP (insecure) mode.
* Docker-engine installed on server.
* Docker-machine installed on host (developer computer).

**obs:** These procedures were tested in a Digital Ocean droplet.

## Step 0: Change the configurations to the desired web address.

* Change the file **nginx.conf**, line 12, **DOMAIN_ADDRESS** to domain address URL.

* Put your email and domain address URL when to run the certbot container:
 * **-m user@exemplo.com**.
 * **-d test.com**.

* Connect to the server with docker-machine.

```
eval $(docker-machine env droplet-name)
```

## Step 01: Run the web server container

After connected in the server with docker-machine, use the command bellow to create a web server image.

```
  docker build -f web_server.dockerfile -t web_server .

Sending build context to Docker daemon  8.192kB
Step 1/2 : FROM nginx:1.15
 ---> cd5239a0906a
Step 2/2 : COPY ./nginx.conf /etc/nginx/nginx.conf
 ---> 192362c0a39c
Successfully built 192362c0a39c
Successfully tagged web_server:latest
```

Then start the web_server with the compose file.
The service will contain two volumes:
* **letsencrypt:** To save the certificate files.
* **acme-challenge:** To the certbot container checks.

```
  docker-compose -f certbot_nginx_compose.yml up --build -d

Creating network "source_default" with the default driver
Creating volume "source_letsencrypt" with default driver
Creating volume "source_acme-challenge" with default driver
Creating web_server ... done
```

After rise the container, verify if everything is working fine:

```
  $ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                                      NAMES
8ad20af9ea93        web_server          "nginx -g 'daemon of…"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   web_server


  $ docker logs web_server

162.216.152.41 - - [12/Jun/2018:02:40:11 +0000] "GET / HTTP/1.0" 200 612 "-" "-"


  $ docker volume ls

DRIVER              VOLUME NAME
local               source_acme-challenge
local               source_letsencrypt
```

And finally check with the web server is accessible by the web. To open the URL in the browser:

```
Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```

##  Step 02: Run the certbot container

The command lines options explanation:

* **-it:** for interactive interface in the container.
* **--rm:** to remove certbot container after run.
* **--volumes-from:** to attach the volumes from other containers.
* **--webroot and --webroot-path:** certbot configurations.
* **--agree-tos:** License agreement.
* **--dry-run:** For tests. Just check the configurations. Don't download certificates.
* **--staging:** For tests. Test configurations and download not valid certificates.
* **-m:** email address from responsible for domain.
* **-d:** the domain address URL.

### Dry-run

Fist it's recommended to run a dry-run procedure.

It will test the configurations, but it won't download any certificates.

```bash
docker run -it --rm \
  --volumes-from web_server \
  certbot/certbot certonly \
  --webroot \
  --webroot-path /tmp/letsencrypt/www \
  --agree-tos \
  --staging \
  --dry-run \
  -m user@exemplo.com \
  -d test.com
```

```
Unable to find image 'certbot/certbot:latest' locally
latest: Pulling from certbot/certbot
ff3a5c916c92: Pull complete
b99d27bed84a: Pull complete
097b40228318: Pull complete
18fbd9159d80: Pull complete
571905616c12: Pull complete
e036a76641b5: Pull complete
01f379d52cce: Pull complete
768fef019f2a: Pull complete
e8c640ed1f90: Pull complete
75793257db0f: Pull complete
Digest: sha256:b5fea88f6ab45e9c718c18e2ecef02332c7e3896659bab4419f82d5795b3ff4f
Status: Downloaded newer image for certbot/certbot:latest
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator webroot, Installer None
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for test.com
Using the webroot path /tmp/letsencrypt/www for all unmatched domains.
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - The dry run was successful.
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
```

### Staging

Once the dry-run perfomed without errors, it's time to staging procedure.

It will test the configurations and to download **non-valid** certificates.

```bash
docker run -it --rm \
  --volumes-from web_server \
  certbot/certbot certonly \
  --webroot \
  --webroot-path /tmp/letsencrypt/www \
  --agree-tos \
  --staging \
  -m user@exemplo.com \
  -d test.com
```

```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator webroot, Installer None
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for test.com
Using the webroot path /tmp/letsencrypt/www for all unmatched domains.
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/test.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/test.com/privkey.pem
   Your cert will expire on 2018-09-10. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"

```

### Complete verification

Finally, to perform the full check.

It will download the oficial certificate to the letsencrypt volume (attached to web_server container).

Be sure before trying for real to avoid to be blocked by the Let's Encrypt service. First run dry-run and staging modes before this step.

```bash
docker run -it --rm \
  --volumes-from web_server \
  certbot/certbot certonly \
  --webroot \
  --webroot-path /tmp/letsencrypt/www \
  --agree-tos \
  -m user@exemplo.com \
  -d test.com
```

```
 Saving debug log to /var/log/letsencrypt/letsencrypt.log
 Plugins selected: Authenticator webroot, Installer None
 Cert not yet due for renewal

 -------------------------------------------------------------------------------
 Would you be willing to share your email address with the Electronic Frontier
 Foundation, a founding partner of the Let's Encrypt project and the non-profit
 organization that develops Certbot? We'd like to send you email about our work
 encrypting the web, EFF news, campaigns, and ways to support digital freedom.
 -------------------------------------------------------------------------------
 (Y)es/(N)o: N
 Cert not yet due for renewal

 You have an existing certificate that has exactly the same domains or certificate name you requested and isn't close to expiry.
 (ref: /etc/letsencrypt/renewal/test.com.conf)

 What would you like to do?
 -------------------------------------------------------------------------------
 1: Keep the existing certificate for now
 2: Renew & replace the cert (limit ~5 per 7 days)
 -------------------------------------------------------------------------------
 Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
 Renewing an existing certificate
 Performing the following challenges:
 http-01 challenge for testing.staging.net.br
 Using the webroot path /tmp/letsencrypt/www for all unmatched domains.
 Waiting for verification...
 Cleaning up challenges

 IMPORTANT NOTES:
  - Congratulations! Your certificate and chain have been saved at:
    /etc/letsencrypt/live/test.com/fullchain.pem
    Your key file has been saved at:
    /etc/letsencrypt/live/test.com/privkey.pem
    Your cert will expire on 2018-09-10. To obtain a new or tweaked
    version of this certificate in the future, simply run certbot
    again. To non-interactively renew *all* of your certificates, run
    "certbot renew"
  - If you like Certbot, please consider supporting our work by:

    Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
    Donating to EFF:                    https://eff.org/donate-le
```

**Obs:**

The existing certificate is the **non-valid** certificate created in the step 02.
If you prefer, it is possible to remove the volume (`docker volume rm source_letsencrypt`) and rerun the web_server service and the certbot container.


## Step 03: Retry the certificate.

If the certbot container runs without errors, the certificates will be available in the letsencrypt volume.

Copy the certificate to developer machine or to other server.

```
docker-machine scp -r server_name:/var/lib/docker/volumes/source_letsencrypt/_data /home/developer/some-path/letsencrypt/

0000_csr-certbot.pem          100%  936     4.6KB/s   00:00    
0001_csr-certbot.pem          100%  936     4.6KB/s   00:00    
test.com.conf                 100%  611     3.0KB/s   00:00    
privkey.pem                   100% 1704     8.4KB/s   00:00    
fullchain.pem                 100% 3818    18.7KB/s   00:00    
cert.pem                      100% 2171    10.7KB/s   00:00    
README                        100%  682     3.4KB/s   00:00    
chain.pem                     100% 1647     8.1KB/s   00:00    
0001_key-certbot.pem          100% 1704     8.4KB/s   00:00    
0000_key-certbot.pem          100% 1704     8.4KB/s   00:00    
meta.json                     100%   72     0.4KB/s   00:00    
private_key.json              100% 1631     8.0KB/s   00:00    
regr.json                     100%  578     2.9KB/s   00:00    
meta.json                     100%   72     0.4KB/s   00:00    
private_key.json              100% 1632     8.0KB/s   00:00    
regr.json                     100%  769     3.8KB/s   00:00    
chain2.pem                    100% 1647     8.1KB/s   00:00    
privkey2.pem                  100% 1704     8.4KB/s   00:00    
fullchain2.pem                100% 3818    18.7KB/s   00:00    
cert2.pem                     100% 2171    10.6KB/s   00:00    
chain1.pem                    100% 1679     8.2KB/s   00:00    
fullchain1.pem                100% 3809    18.7KB/s   00:00    
cert1.pem                     100% 2130    10.5KB/s   00:00    
privkey1.pem                  100% 1704     8.4KB/s   00:00
```

or

```
docker-machine scp -r server_name:/var/lib/docker/volumes/source_letsencrypt/_data another_server_name:/some-path/letsencrypt/

[...]
```
