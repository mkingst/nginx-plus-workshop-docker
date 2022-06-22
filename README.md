# Using NGINX Plus as a Wordpress Load Balancer in Docker

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

### 1. Create NGINX Plus Docker Image

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

You now have an NGINX Plus container running. 


### 4. Access NGINX Plus from a browser on localhost:80. 

![image](https://user-images.githubusercontent.com/44472403/175025402-208d0810-0c6a-4bb9-aa4f-5a6392c6cc38.png)

## Configure NGINX as a Load Balancer for two Wordpress Sites

### Execute a shell into the NGINX container

``` 
root@ip-172-31-6-122:~/nginx_docker# docker exec -it mynginxplus bash
root@28d88ee569b8:/#
```

### Go to the core NGINX configuration directory

```root@28d88ee569b8:/# cd /etc/nginx/conf.d/```

### Remove the default configuration (Welcome to NGINX Page) and create a new configuration file called ***wordpress.conf.

```
root@28d88ee569b8:/etc/nginx/conf.d# rm default.conf

root@28d88ee569b8:/etc/nginx/conf.d# vi wordpress.conf
```

### Use the following configuration:

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

- The "upstream" block is a load balancing pool that includes both Wordpress backends (172.17.0.1 is the default IP for the Docker host)
- The "keepalive" directive sets the maximum number of idle keepalive connections to upstream servers that are preserved in the cache of each worker process. 
- The NGINX virtual server is listening in port 80
- We have access and error logs configured
- Added a health check to ensure traffic is not sent to a backend that has no HTTP response
- The $upstream_addr variable includes the IP of the backend NGINX is proxying to (upstream IP). 
- Added a X-Upstream-Add header to see the backend IP in your browser

### Reload NGINX

```
root@28d88ee569b8:/etc/nginx/conf.d# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

root@28d88ee569b8:/etc/nginx/conf.d# nginx -s reload
2022/06/22 12:22:55 [notice] 83#83: signal process started
```

### Access NGINX in your browser 

NGINX is now acting as a proxy and load balancer for both Wordpress backends. Refresh the page continuously to see round-robin load balancing. 

![image](https://user-images.githubusercontent.com/44472403/175027676-f1fac771-11b5-4d46-8736-b3dd69dc0b56.png)

## Enhanced NGINX Configuration (full config avaiable in repo)

### Enable the NGINX Plus Dashboard and API

```
    location /dashboard.html {
        root /usr/share/nginx/html;
    }

    location /api {
        api write=on;
        #auth_basic "Login Required";
        #auth_basic_user_file /etc/nginx/.htpasswd;
    }
```

![image](https://user-images.githubusercontent.com/44472403/175037072-f77db5aa-dd3d-4f29-b7a3-a51516186bae.png)

https://www.nginx.com/products/nginx/live-activity-monitoring/ 

### Add TLS and HTTP2

```
server {

        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
        server_name localhost;
        ssl_certificate /etc/nginx/ssl/public.pem;
        ssl_certificate_key /etc/nginx/ssl/private.key;
        ssl_protocols TLSv1.2;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
        ssl_prefer_server_ciphers on;
        
        ..,
      }
```
https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/ 

### Add Caching (cache all 200 responses)

```

proxy_cache_path /tmp/cache keys_zone=mycache:10m levels=1:2 inactive=3600s max_size=100m;

server {
        listen 80 default_server;
        server_name localhost;

        access_log /var/log/nginx/proxy.access.log;
        error_log /var/log/nginx/proxy.error.log info;
        
        proxy_cache mycache;
        proxy_cache_valid 200 1m;
        proxy_cache_lock on;
        proxy_cache_use_stale http_500 http_502 http_503 http_504;

    location / {
        proxy_set_header Connection "";
        proxy_set_header Host $upstream_addr;
        add_header X-Upstream-Add $upstream_addr;
        health_check;
        proxy_pass http://wordpress-pool;
   }

}
```

Note: Added HTTP Header to see if the request is hitting NGINX Cache

![image](https://user-images.githubusercontent.com/44472403/175038031-eb0b88bd-20e7-4a6d-875e-a7891ee040d8.png)

https://docs.nginx.com/nginx/admin-guide/content-cache/content-caching/

### Change the Load Balancing Method to least_time and add Session Persistence

```
upstream wordpress-pool {
        
        zone wp-upstreams 64k;

        least_time header;
        server 172.17.0.1:11081;
        server 172.17.0.1:11082;

        keepalive 20;

	sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/#choosing-a-load-balancing-method
https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/#enabling-session-persistence


### Additional Links

Installing NGINX Plus: https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/

Full NGINX Directive Index: http://nginx.org/en/docs/dirindex.html 

NGINX Administration Guide: https://docs.nginx.com

