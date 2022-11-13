# How to create asp.net core (.Net 6) in docker with nginx in linux

Creating two docker containers (microservices):
1. First with your application Asp.Net Core
2. Secound with nginx server that will be a reverse server

## Simple manual way

1. Create a docker diretory `mkdir docker`, `cd docker`
2. Create directories for your application `mkdir asp-www` and for nginx `mkdir nginx`
### Asp.Net application
4. Go to application folder `cd asp-www`
5. Copy published asp.net core appication data folder
6. Create new file Dockerfile `touch Dockerfile` 
7. You should have something like that:

![image](https://user-images.githubusercontent.com/54003204/201540226-60880aaf-bcad-4e3a-b9da-b3c92e36705d.png)

8. Copy content to `Dockerfile` e.g. `nano Dockerfile`:

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

9. Build docker image with tag (-t) 'asptest' `sudo docker build -t asptest .`
10. Run docker image 'asptest' and name container 'asptest' (--name asptest) `sudo docker run --rm -it --name asptest asptest`

![image](https://user-images.githubusercontent.com/54003204/201541526-73e48979-d590-4f9e-a154-895013225b1c.png)

### Nginx server (reverse proxy)
11. Go to filder `cd ~/docker/nginx`
12. Create files `touch Dockerfile` and `touch nginx.conf`
13. Edit nginx.conf `nano nginx.conf` and insert:

```
worker_processes 1;
events { worker_connections 1024; }
http {
    sendfile on;

    upstream asptest { #container name of yourapplication (asptest)
	    server asptest:80; # the port for your asp.net application
    }
    
server {
	listen 80;
	server_name app.example.pl; # your domain name - you should change it
	location / {
	proxy_pass         http://asptest; # container name of your application (asptest)
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

14. Edit Dockerfile `Dockerfile` and insert:

```
FROM nginx

COPY nginx.conf /etc/nginx/nginx.conf
```

15. You should have files:

![image](https://user-images.githubusercontent.com/54003204/201540733-a75caede-bef4-4f71-a04a-01239b0ab772.png)

16. Build docker image for nginx `sudo docker build -t nginx-test .`
17. Run docker image 'nginx-test' linked with our first container name (--link asptest) and map server port 80 to container 80 
`sudo docker run --rm -it -p 80:80 --link asptest  nginx-test`

![image](https://user-images.githubusercontent.com/54003204/201541700-e41698c3-caca-4bbc-9c5f-7340af7fadd9.png)

18. Open web browser and open your server ip port. In browser you should see your application and in terminals something like this:

![image](https://user-images.githubusercontent.com/54003204/201541797-d7e00618-87cb-4be9-9aaa-2f39517efa21.png)


