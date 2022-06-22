# Using NGINX Plus as a Load Balancer in Docker

## Requirements

- Laptop / Linux host / VM

- Internet connection

- Docker installed
  https://docs.docker.com/get-docker/

- NGINX installed
  https://docs.nginx.com/nginx/admin-guide/installing-nginx/


  ``` $ nginx -v ```

## Deploy Two Wordpress Containers in Docker

### Create Docker Bridge Network
```docker network create --driver bridge my_network```
### Create MySQL Database Container
``` 
docker run --name mysql-wp --hostname mysql-wp --network=my_network -e MYSQL_ROOT_PASSWORD=ChangeMe -d mysql:5.7; 
```
### Create Wordpress Databases 
```
docker exec -it mysql-wp mysql -pChangeMe;
CREATE DATABASE WP1;
CREATE DATABASE WP2;
exit;
\q
```

### Deploy Wordpress Containers
```
docker run -d --link mysql-wp:mysql -p 11081:80 --name wp1 --hostname wp1 --network=my_network --ip 172.18.0.7 -e WORDPRESS_DB_HOST=mysql-wp -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=ChangeMe -e WORDPRESS_DB_NAME=WP1 wordpress:4.8;

docker run -d --link mysql-wp:mysql -p 11082:80 --name wp2 --hostname wp2 --network=my_network --ip 172.18.0.8 -e WORDPRESS_DB_HOST=mysql-wp -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=ChangeMe -e WORDPRESS_DB_NAME=WP2 wordpress:4.8
```

### Configure Wordpress

Access Wordpress on localhost:11081 and localhost:11082 and finish configuration
