# Nginx Docker-compose

For configuration see: [docker/README](../docker/README.md)

```text
version: '3.1'
services:
  web:
    image: nginx
    volumes:
     - ./templates:/etc/nginx/templates
    ports:
     - "8080:80"
    environment:
     - NGINX_HOST=foobar.com
     - NGINX_PORT=80
```