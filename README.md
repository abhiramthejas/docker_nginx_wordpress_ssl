Here we will build a multi-container WordPress installation. Which includes a MySQL database, an Nginx web server, and WordPress containers. We will also add SSL to the domain by obtaining TLS/SSL certificates with Let’s Encrypt


![diagram docker wordpress](https://user-images.githubusercontent.com/17767960/195329833-e336939c-cdf5-4ff5-962a-66d8444644e6.png)

Prerequisites

*Need server with Ubuntu 18.04 or above.
*Need to install Docker, you can use this for installing Docker .

*Need to install Docker-compose, you can use this for installing Docker-compose .

*Purchase a domain name or point you domain name to server IP. You can use this to purchase new domain_name.

Need to point:

*An A record with abhiramthejas.online pointing to your server’s public IP address.

*An A record with www.abhiramthejas.online pointing to your server’s public IP address. (Replace abhiramthejas.online with your domain name in this entire git)

Procedure
1. To configure Nginx web server

First, create a project directory for your project setup called project and navigate to it:
```
mkdir project && cd project
vi nginx-conf/nginx.conf
```
Edit nginx.conf

```
server {
        listen 80;
        listen [::]:80;

        server_name abhiramthejas.online www.abhiramthejas.online;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }
        
        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```

2. Create Environmental variables

```
vi .env
```
```
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_USER=your_wordpress_database_user
MYSQL_PASSWORD=your_wordpress_database_password
```

3. Create Docker compose yml file 

```
vi docker-compose.yml
```

Here we are using mysql:8 image with conatainer name "database" and we use wordpress image 5.1.1-fpm-alpine with container name "wordpress", this image have php-fpm enabled. Also, we are using nginx:1.15.12-alpine for webserver/Nginx image with container name "webserver". We define certbot after the webserver container. All of these containers are added in same network "wpnet".

```
version: '3'

services:
  database:
    image: mysql:8.0
    container_name: database
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - db-data:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - wpnet

  wordpress:
    depends_on: 
      - database
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=database:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wp-data:/var/www/html
    networks:
      - wpnet

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    ports:
      - "80:80"
    volumes:
      - wp-data:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - ./logs/nginx:/var/log/nginx/
      - certbot:/etc/letsencrypt
    networks:
      - wpnet

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt
      - wp-data:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email mail@abhiramthejas.online --agree-tos --no-eff-email --staging -d abhiramthejas.online -d www.abhiramthejas.online

volumes:
  certbot:
  wp-data:
  db-data:

networks:
  wpnet:
    driver: bridge
``` 
Initiate docker container using the docker-compose command

```
docker-compose config
docker-compose up -d
```
After creating container make sure SSL certificate geenrated 

```
docker-compose exec webserver ls -l /etc/letsencrypt/live/
```
```
# docker-compose exec webserver ls -l /etc/letsencrypt/live/
total 8
-rw-r--r--    1 root     root           740 Oct 12 10:55 README
drwxr-xr-x    2 root     root          4096 Oct 12 11:08 abhiramthejas.online
```

Then we can change the certbot --staging flag to --force-renewal flag in docker compose file to request a new certificate with the same domains as an existing certificate and run
```
docker-compose up --force-recreate --no-deps certbot
```
To recreate the certbot container include the --no-deps option to tell compose that it can skip starting the webserver service, since it is already running.

We can then download and place the recommended Nginx security parameteres after stoping the nginx container

```
docker-compose stop webserver
curl -sSLo nginx-conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
```

Replace nginx conf with SSL settings

```
server {
        listen 80;
        listen [::]:80;

        server_name abhiramthejas.online www.abhiramthejas.online;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name abhiramthejas.online www.abhiramthejas.online;

        index index.php index.html index.htm;

        root /var/www/html;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/abhiramthejas.online/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/abhiramthejas.online/privkey.pem;

        include /etc/nginx/conf.d/options-ssl-nginx.conf;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
        # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # enable strict transport security only if you understand the implications

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }
        
        location = /favicon.ico { 
                log_not_found off; access_log off; 
        }
        location = /robots.txt { 
                log_not_found off; access_log off; allow all; 
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}
```

Modify docker compose file with SSL port open for nginx 

```
version: '3'

services:
  database:
    image: mysql:8.0
    container_name: database
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - db-data:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - wpnet

  wordpress:
    depends_on: 
      - database
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=database:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wp-data:/var/www/html
    networks:
      - wpnet

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - wp-data:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - ./logs/nginx:/var/log/nginx/
      - certbot:/etc/letsencrypt
    networks:
      - wpnet

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt
      - wp-data:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email mail@abhiramthejas.online --agree-tos --no-eff-email --force-renewal -d abhiramthejas.online -d www.abhiramthejas.online

volumes:
  certbot:
  wp-data:
  db-data:

networks:
  wpnet:
    driver: bridge
```

Recreate the nginx container

```
docker-compose up -d --force-recreate --no-deps webserver
```

Now we can access the domain usign the SSL url

![docker_compose wordpress](https://user-images.githubusercontent.com/17767960/195335611-96151f88-7e8e-4efc-9a01-8e758c90395a.jpg)


