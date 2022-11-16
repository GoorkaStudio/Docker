# Asp.Net Core (.Net 6) web application with PostgreSQL and Nginx

Our system is build from two subsystem: 
- main www serwer (nginx) witch works as reverse proxy for services/applications, with let's encrypt ssl certificat
- services/applications - run on container with asp.net
The system is running on subdomain 'app.MYDOMAIN.com' (on MYDOMAIN.com we have static content from another server)

Everything runs on it's own container. Services might be deployed independenty, but for services to run on https://app.MYDOMAIN.com/MY-APP-NAME they must be configureted in main www server (nginx) or they might by run with theyr own nginx (but on another port without ssl)

*If you have a problem start with simple version https://github.com/GoorkaStudio/Docker/blob/main/asp-net-nginx-linux-simple.md

## Stage 1 - Main serwer www Nginx
1. Create folder structure

![image](https://user-images.githubusercontent.com/54003204/202036462-49b6d61b-d87c-4fc1-88e8-707ff8910642.png)


2. Create directory `~/docker/main-server-www/nginx`
3. Create file `~/docker/main-server-www/nginx/nginx.conf` with content and change app.MYDOMAIN to you own:
```
worker_processes 1;
events { worker_connections 1024; }
http {
    sendfile on;

server {
        listen 80;
        server_name app.MYDOMAIN.com;

	location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}
```
This config file is for run and configure Lets Encrypt with Certbot [https://www.programonaut.com/setup-ssl-with-docker-nginx-and-lets-encrypt/]

4. Create file `~/docker/main-server-www/nginx/Dockerfile` 
```
FROM nginx

COPY nginx.conf /etc/nginx/nginx.conf
```
5. Create file `~/docker/main-server-www/docker-compose.yml`
```
version: "3.7"

services:
# main www server - nginx
  nginx-main:
    container_name: nginx-main
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./certbot/conf:/etc/letsencrypt 
      - ./certbot/www:/var/www/certbot
    image: nginx-main
    restart: always
    build:
      context: ./nginx

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes: 
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    command: certonly --webroot -w /var/www/certbot --force-renewal --email MYEMAIL@gmail.com -d app.MYDOMAIN.com --agree-tos

```

6. In folder `~/docker/main-server-www/` run `sudo docker compose build` and `sudo docker compose up nginx-main` - this should start nginx and map `app.MYDOMAIN.com/.well-known/acme-challenge/` to folder `./certbot/www`

7. Run `sudo docker compose up certbot` - this should create folder `~/docker/main-server-www/certbot` with certificates

8. Stop nginx `sudo docker compose down nginx-main`
9. Change nginx config - now we will be use ssl - edit file `~/docker/main-server-www/nginx/nginx.conf` with content and change app.MYDOMAIN to you onw:
```
worker_processes 1;
events { worker_connections 1024; }
http {
    sendfile on;

#    upstream hub-language-tests-aspweb { #container name of yourapplication (asptest)
#	    server hub-language-tests-aspweb:80; # the port for your asp.net application
#    }

# always redirect to https
    server {
        listen 80;
        server_name app.MYDOMAIN.com;
        return 301 https://$host$request_uri;
    }

server {
        listen 443 ssl http2;
        # use the certificates
        ssl_certificate     /etc/letsencrypt/live/app.MYDOMAIN.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/app.MYDOMIAN.com/privkey.pem;
        server_name app.MYDOMAIN.com;
        root /var/www/html;
        index index.php index.html index.htm;       

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

	location /hub/test-aspweb {
	    rewrite 	       /hub/language-tests(.*) $1 break;
	    proxy_pass         http://hub-language-tests-aspweb; # container name of your application (asptest)
	    proxy_http_version 1.1;
	    proxy_set_header   Upgrade $http_upgrade;
	    proxy_set_header   Connection keep-alive;
	    proxy_set_header   Host $host;
	    proxy_cache_bypass $http_upgrade;
	    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_set_header   X-Forwarded-Proto $scheme;
	}
 }
}
```

10. In folder `~/docker/main-server-www/` run `sudo docker compose build` and `sudo docker compose up nginx-main`




## Stage 2 - Asp.Net application
1. Create folder `~/docker/apps-hub/language-tests` and in this folder create folders: `asp-web-app` `data`

![image](https://user-images.githubusercontent.com/54003204/202049528-38781a92-7c1b-4959-beba-310dfa256a7d.png)

3. Copy your application (publish) to `asp-web-app` and  create file `Dockerfile` 
```
# base docker image - it's orginall MS image with Asp.Net installed
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base 

# creating folder 'app' in image
WORKDIR /app

# by default export 80 port on with runs your application - this is container local port
EXPOSE 80 

#coping all your application files
COPY ./ ./ 

#application that will be run when container starte (change AngielskiAnkieta.dll to your application file name)
ENTRYPOINT ["dotnet", "AngielskiAnkieta.dll"] 
```

4. In folder `~/docker/apps-hub/language-tests/data` create folders `db-postgres` and `web-app-data`

![image](https://user-images.githubusercontent.com/54003204/202050865-262fa11f-0164-4268-be4c-3966011b27d5.png)

5. In folder `~/docker/apps-hub` create file `docker-compose.yml`
```
version: "3.7"

services:
# HUB applications

# language-tests
  hub-language-tests-postgres:
    container_name: hub-language-tests-postgres
    image: "postgres"
#    ports:
#      - 5432:5432  #if you want to access to db from outside (for debug/dev)
    volumes:
      - ./apps-hub/language-tests/data/db-postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: YOURUSERNAME
      POSTGRES_PASSWORD: YOURPASSWORD
    restart: always

  hub-language-tests-aspweb:
    depends_on:
      - hub-language-tests-postgres
    container_name: hub-language-tests-aspweb
    image: asptest
    volumes:
      - ./apps-hub/language-tests/data/web-app-data:/app/data
    restart: always
    build:
      context: ./apps-hub/language-tests/asp-web-app
```






