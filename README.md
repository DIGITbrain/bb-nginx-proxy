# General

## Deployment type

docker

## Image

Based on official image on Docker Hub: https://hub.docker.com/_/nginx

## Licence

Nginx license: http://nginx.org/LICENSE

## Version

1.19.7

## Description

A Nginx reverse proxy server is a type of proxy server that typically sits behind the firewall in a private network and directs client requests to the appropriate backend server. A reverse proxy provides an additional level of abstraction and control to ensure the smooth flow of network traffic between clients and servers.Common uses for a reverse proxy server include: Load balancing, Web acceleration Security and anonymity.

# Deployment

A. Web server example:

```sh
docker run -d --rm \
        --name nginx \
        -p 80:80 \
        -p 443:443 \
        -v $HOME/nginx/data:/usr/share/nginx/html:ro \
        nginx:1.19.7
```

See [1] for more details.

> Note that, `:ro` tag means that the data is only readable from the nginx. If you want to write data, remove `:ro` tag.

<br/>

B. Proxy and Load balancer example:

```sh
docker run -d --rm
        --name nginx \
        -v $HOME/nginx/config/nginx.conf:/etc/nginx/nginx.conf:ro \
        nginx:1.19.7
```

See [2] [3] for more details.

## Parameters

|Name|Value|Description|
|-|-|-|
|Ports|`-p 80:80`<br/>`-p 443:443`| HTTP port<br/> HTTPS port |
|Volumes|`-v $HOME/nginx/data:/usr/share/nginx/html:ro`<br/>`-v $HOME/nginx/config/nginx.conf:/etc/nginx/nginx.conf:ro` | Store the required files and data <br/> Nginx configuration file |

<br/>

Available options:
- use template files: ```*.template``` and use notation: ```${SOME_VAR}```
- define ```environment``` variable as you can see below
- mount template files to: ```/etc/nginx/templates``` (e.g. -v $PWD/templates:/etc/nginx/templates)
- vars substituted into template, .template suffix removed, then copied to output dir

Available environment variables:
- `NGINX_ENVSUBST_TEMPLATE_DIR`: defaults to /etc/nginx/templates
- `NGINX_ENVSUBST_TEMPLATE_SUFFIX`: defaults to .template
- `NGINX_ENVSUBST_OUTPUT_DIR`: defaults to /etc/nginx/conf.d

For additional configuration edit `/etc/nginx/nginx.conf`. See [4] for more details.

## Testing

A sample config file (```nginx.conf```):
```nginx
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

# TLS

SSL Reverse Proxy:

|Port|Description|Configuration setting|
|-|-|-|
|443| SSL port | `listen` |

See [5] for more details.

To create server certificates (`ca.crt`, `server.key`, `server.crt`), execute the following, replace the subject (value of `-subj` parameter) as needed:

```sh
mkdir certs

# Create ca
openssl genrsa -out certs/ca.key 4096
openssl req -x509 -new -nodes -sha256 \
        -key certs/ca.key \
        -days 3650 \
        -subj '/O=Nginx Test/CN=Certificate Authority' \
        -out certs/ca.crt

# Create server certificates: server.key, server.crt
openssl genrsa -out certs/server.key 2048
openssl req -new -sha256 \
        -key certs/server.key \
        -subj '/O=Nginx Test/CN=www.example.com' | openssl x509 \
        -req -sha256 -CA certs/ca.crt \
        -CAkey certs/ca.key \
        -CAserial certs/ca.srl \
        -CAcreateserial \
        -days 365 \
        -out certs/server.crt

# Create client certificate: client.key and client.crt singed by ca.crt
openssl genrsa -out certs/client.key 2048
openssl req -new -sha256 \
        -key certs/client.key \
        -subj '/O=Nginx Test/CN=Client' | openssl x509 \
        -req -sha256 -CA certs/ca.crt \
        -CAkey certs/ca.key \
        -days 365 \
        -out certs/client.crt
```

### Deployment

Execute the following (and replace parameters as needed):

```sh
docker run -d --rm \
        --name nginx \
        -v $PWD/nginx.conf:/etc/nginx/nginx.conf \
        -v $PWD/certs/server.crt:/etc/nginx/server.crt \
        -v $PWD/certs/server.key:/etc/nginx/server.key \
        -p 443:443 \
        nginx:1.19.7
```

### Testing

Direct your browser at:

- https://`<host_ip>` (HTTPS site with self signed certs)

Run curl with client authentication:

```sh
curl --insecure \
        --cert certs/client.crt \
        --key certs/client.key \
        https://localhost:443
```

# References

[1] https://docs.nginx.com/nginx/admin-guide/web-server/web-server/

[2] https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/

[3] https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/

[4] https://www.nginx.com/resources/wiki/start/topics/examples/full

[5] https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http

