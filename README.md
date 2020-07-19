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
http {
  # Redirect HTTP to HTTPS
  server {
    listen 80;
    server_name www.gary.ascuy.me gary.ascuy.me;
    return 301 https://$host$request_uri;
  }

  server {
    server_name gary.ascuy.me;
    
    ssl_certificate      /etc/letsencrypt/live/gary.ascuy.me/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/gary.ascuy.me/privkey.pem;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";  
    ssl_trusted_certificate /etc/letsencrypt/live/gary.ascuy.me/fullchain.pem;

    location / {
      root   /usr/share/nginx/html;
      # proxy_pass http://api.domain.local:3666;
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