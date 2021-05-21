# IMAGE SOURCE

Official image on __Docker Hub__: https://hub.docker.com/_/nginx

# Licence

Nginx license: http://nginx.org/LICENSE

# Version

1.19.7

# DEPLOYMENT


__Web server__:
> docker run --name nginx -v $PWD/mycontent:/usr/share/nginx/html:ro -d nginx:1.19.7

Notes:
- container port 80 is opened
- see: https://docs.nginx.com/nginx/admin-guide/web-server/web-server/


__Proxy/load balancer__:

> docker run --name nginx -v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro -d nginx:1.19.7

OPTIONS:
- use template files: __*.template__ and use notation: __${SOME_VAR}__
- define __SOME_VAR__ as __environment__ variable
- mount template files to: __/etc/nginx/templates__ (e.g. -v $PWD/templates:/etc/nginx/templates)
- vars substituted into template, .template suffix removed, then copied to output dir
- NGINX_ENVSUBST_TEMPLATE_DIR: defaults to /etc/nginx/templates
- NGINX_ENVSUBST_TEMPLATE_SUFFIX: defaults to .template
- NGINX_ENVSUBST_OUTPUT_DIR: defaults to /etc/nginx/conf.d


Further details and configuration: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/



# TLS

__SSL Reverse Proxy__

Create server certificates: __server.key__, __server.crt__.

```text
$ mkdir certs
$ openssl genrsa -out certs/ca.key 4096
$ openssl req -x509 -new -nodes -sha256 -key certs/ca.key -days 3650 -subj '/O=Nginx Test/CN=Certificate Authority' -out certs/ca.crt
$ openssl genrsa -out certs/server.key 2048
$ openssl req -new -sha256 -key certs/server.key -subj '/O=Nginx Test/CN=www.example.com' | openssl x509 -req -sha256 -CA certs/ca.crt -CAkey certs/ca.key -CAserial certs/ca.srl -CAcreateserial -days 365 -out certs/server.crt
```

 Create config file: __nginx.conf__:
```text
events {}
http {
  server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     server.crt;
    ssl_certificate_key server.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    location / {
        proxy_pass          https://www.sztaki.hu/;
    }
  }
}
```

Run:
> docker run --rm --name nginx -v $PWD/nginx.conf:/etc/nginx/nginx.conf -v $PWD/certs/server.crt:/etc/nginx/server.crt -v $PWD/certs/server.key:/etc/nginx/server.key -p 443:443 -d nginx:1.19.7

## TEST

> Open https://host, proceed unsafe, see sztaki web page with https://host URL.


# TLS mutual authentication

Set __ssl_verify_client__ and __ssl_client_certificate__ in config file:


```text
events {}
http {
  server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_verify_client   on;
    ssl_client_certificate /etc/nginx/ca.crt;
    ssl_certificate     server.crt;
    ssl_certificate_key server.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    location / {
        proxy_pass          https://www.sztaki.hu/;
    }
  }
}
```
Run: 

> docker run --name nginx -v $PWD/nginx.conf:/etc/nginx/nginx.conf:ro -v $PWD/certs/server.crt:/etc/nginx/server.crt -v $PWD/certs/server.key:/etc/nginx/server.key -v $PWD/certs/ca.crt:/etc/nginx/__ca.crt__ -p 443:443 -d nginx:1.19.7

## TEST

Create client certificate: __client.key__ and __client.crt__ singed by *ca.crt*:
```
$ openssl genrsa -out certs/client.key 2048
$ openssl req -new -sha256 -key certs/client.key -subj '/O=Nginx Test/CN=Client' | openssl x509 -req -sha256 -CA certs/ca.crt -CAkey certs/ca.key -days 365 -out certs/client.crt
```

Run *curl* with client authentication: 

 > curl --insecure --cert certs/client.crt --key certs/client.key https://localhost:443
