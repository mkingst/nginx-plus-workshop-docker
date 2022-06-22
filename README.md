# Using NGINX Plus as a Load Balancer in Docker

## Requirements

- Laptop / Linux host / VM

- Internet connection

- Docker installed
  https://docs.docker.com/get-docker/

- NGINX Plus Cert and Key
  https://www.nginx.com/free-trial-request/

## Deploy Two Wordpress Containers in Docker

### Create MySQL Database Container
``` 
docker run --name mysql-wp --hostname mysql-wp -e MYSQL_ROOT_PASSWORD=ChangeMe -d mysql:5.7; 
```
**Note: Wait 10-20 seconds for the container to start**

### Create Wordpress Databases 
```
docker exec -it mysql-wp mysql -u root -pChangeMe;
CREATE DATABASE WP1;
CREATE DATABASE WP2;
exit;
\q
```

### Deploy Wordpress Containers
```
docker run -d --link mysql-wp:mysql -p 11081:80 --name wp1 --hostname wp1 -e WORDPRESS_DB_HOST=mysql-wp -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=ChangeMe -e WORDPRESS_DB_NAME=WP1 wordpress:4.8;

docker run -d --link mysql-wp:mysql -p 11082:80 --name wp2 --hostname wp2 -e WORDPRESS_DB_HOST=mysql-wp -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=ChangeMe -e WORDPRESS_DB_NAME=WP2 wordpress:4.8;
```

### Configure Wordpress

Access Wordpress on localhost:11081 and localhost:11082 and finish configuration

***Wordpress 1***

![image](https://user-images.githubusercontent.com/44472403/175015731-0e84a348-5e43-4709-ac7e-c8eeb8d67c31.png)

***Wordpress 2***

![image](https://user-images.githubusercontent.com/44472403/175015927-cbfa3214-6e1d-43a5-bf30-a5ac83c37ef2.png)

### Access both Wordpress sites using your Browser, on port 11081 and 11082:

![image](https://user-images.githubusercontent.com/44472403/175024994-5cd38621-4a74-4a15-a1b7-14e0f4682423.png)

![image](https://user-images.githubusercontent.com/44472403/175025240-6c3c2083-3cf0-4ae7-aa17-0da92dab0519.png)

## Install NGINX in Docker

Docker can also be used with NGINX Plus. The difference between using Docker with NGINX Open Source is that you first need to create an NGINX Plus image, because as a commercial offering NGINX Plus is not available at Docker Hub.

####1. Creating NGINX Plus Docker Image

Download your nginx-repo.crt and nginx-repo.key files. For a trial of NGINX Plus, the files are provided with your trial package.

Copy the files to the directory where the included Dockerfile is located.

Create a Docker image, for example, nginxplus (note the final period in the command).

```$ DOCKER_BUILDKIT=1 docker build  --no-cache --secret id=nginx-key,src=nginx-repo.key --secret id=nginx-crt,src=nginx-repo.crt -t nginxplus .```

The --no-cache option tells Docker to build the image from scratch and ensures the installation of the latest version of NGINX Plus. If the Dockerfile was previously used to build an image without the --no-cache option, the new image uses the version of NGINX Plus from the previously built image from the Docker cache.

### 2. Verify that the nginxplus image was created successfully with the docker images command:
```
$ docker images nginxplus
REPOSITORY  TAG     IMAGE ID      CREATED        SIZE
nginxplus   latest  ef2bf65931cf  6 seconds ago  91.2 MB
```

### 3. Create a container based on this image, for example, mynginxplus container:

```$ docker run --name mynginxplus -p 80:80 -d nginxplus```

You not have an NGINX Plus container running. 

### 4. Access NGINX Plus from a browser on localhost:80. 

![image](https://user-images.githubusercontent.com/44472403/175025402-208d0810-0c6a-4bb9-aa4f-5a6392c6cc38.png)


## Configure NGINX as a Load Balancer for two Wordpress Sites

### Execute a shell into the NGINX container

``` root@ip-172-31-6-122:~/nginx_docker# docker exec -it mynginxplus bash
    root@28d88ee569b8:/#
```

### Go to the core NGINX configuration directory

```root@28d88ee569b8:/# cd /etc/nginx/conf.d/```

### Remove the default configuration (Welcome to NGINX Page) and create a new configuration file called ***wordpress.conf.

```
root@28d88ee569b8:/etc/nginx/conf.d# rm default.conf
root@28d88ee569b8:/etc/nginx/conf.d# vi wordpress.conf
```

### Use wordpress.conf included in this repo

```
upstream wordpress-pool {
        zone wp-upstreams 64k;
        server 172.17.0.1:11081;
        server 172.17.0.1:11082;
        keepalive 20;
}

server {
        listen 80 default_server;
        server_name localhost;

        access_log /var/log/nginx/proxy.access.log;
        error_log /var/log/nginx/proxy.error.log info;

    location / {
        proxy_set_header Connection "";
        proxy_set_header Host $upstream_addr;
        add_header X-Upstream-Add $upstream_addr;
        health_check;
        proxy_pass http://wordpress-pool;
   }

}
```
