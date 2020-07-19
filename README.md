# Docker + Free SSL/TLS Certs (Let's Encrypt)

### Environment 

- Ubuntu 18.04.3 (LTS) x64
- Docker ( `curl -fsSL https://get.docker.com | sh` )
- Config Folder ( we will use /root/nginx in this example )

### Generate Let's Encrypt Certs

```sh
$ docker run --rm -p 80:80 -p 443:443 \
    -v /root/nginx/letsencrypt:/etc/letsencrypt \
    certbot/certbot certonly -d gary.ascuy.me \
    --standalone -m gary.ascuy@gmail.com --agree-tos
```

### Generate Diffie-Hellman Parameter

```sh
$ openssl dhparam -out /root/nginx/dhparam.pem 4096
```

### Config for Nginx 
 
```
# Basic Nginx Config for HTTPS
events {
  worker_connections  4096;
}

http {
  map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
  }

  # Redirect HTTP to HTTPS
  server {
    listen 80;
    server_name www.gary.ascuy.me gary.ascuy.me;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl default deferred;
    server_name gary.ascuy.me;
    
    ssl_certificate      /etc/letsencrypt/live/gary.ascuy.me/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/gary.ascuy.me/privkey.pem;

    # Improve HTTPS performance with session resumption
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 5m;

    # Enable server-side protection against BEAST attacks
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;

    # Disable SSLv3
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

    # Diffie-Hellman parameter for DHE ciphersuites
    # $ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # Enable HSTS (https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security)
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";  

    # Enable OCSP stapling (http://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox)
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/gary.ascuy.me/fullchain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Required for LE certificate enrollment using certbot
    # location '/.well-known/acme-challenge' {
    #   default_type "text/plain";
    #   root /var/www/html;
    # }

    location / {
      root   /usr/share/nginx/html;

      # Proxy Config
      # proxy_pass http://api.domain.local:3666;
      # proxy_http_version 1.1;
      # proxy_set_header Upgrade $http_upgrade;
      # proxy_set_header Connection $connection_upgrade;
      # proxy_set_header X-Forwarded-For $remote_addr;
      # proxy_set_header Host $host;
    }
  }
}
```

### 

```sh
$ docker run --restart always -d -p 80:80 -p 443:443 \
    -v /root/nginx/letsencrypt:/etc/letsencrypt \
    -v /root/nginx/dhparam.pem:/etc/ssl/certs/dhparam.pem \
    -v /root/nginx/nginx.conf:/etc/nginx/nginx.conf \
    --name proxy \
    nginx:alpine
```