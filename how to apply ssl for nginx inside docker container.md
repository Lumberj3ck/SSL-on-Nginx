[Source](https://mindsers.blog/en/post/https-using-nginx-certbot-docker/)

```nginx
upstream backend_mangalove{
    server backend_mangalove:8000;
}

server {
    listen 80;

    server_name mangalove.site www.mangalove.site;
    location / {
        proxy_pass http://backend_mangalove;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;                                                         proxy_redirect off;
    }

	location /static/ {                                                                      root /home/app;
    }

    location /imgs/ {                                                                        root /home/app/;
    }

}
 
```
```code
  mangalove_nginx:
    build: ./nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - static_files:/home/app/static
      - imgs:/home/app/imgs
      - ./certbot/www:/var/www/certbot/:ro
      - ./certbot/conf/:/etc/nginx/ssl/:ro

  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
      - ./certbot/conf/:/etc/letsencrypt/:rw
```
We are starting from nginx complete configuration.

**This nginx conf listens 80 port and proxies all the incoming traffic to upstream gunicorn**

Docker compose maps ports 80 and 443 for the ssl
- ***./certbot/www:/var/www/certbot/:ro   this is folder where certbot saves files using this nginx in container and certbot can communicate*** 

**Now what we need is just to transfer all request from http to https** 

```nginx
upstream backend_mangalove{
    server backend_mangalove:8000;
}

server {
    listen 80;

    server_name mangalove.site www.mangalove.site;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://mangalove.site$request_uri;
    }
}

```

**Here we just listening for path required by certbot and add location directive which basically encaptures every request and redirecting them into https** 

Now we need to add cerbot to our docker compose file 
```docker
  certbot:                                                                               image: certbot/certbot:latest
	volumes:                                                                               - ./certbot/www/:/var/www/certbot/:rw                                                - ./certbot/conf/:/etc/letsencrypt/:rw
```

**Now you can test that everything is working** ( Make sure that your nginx container is up)

```bash
sudo docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d example.org
```

Since we don't have any errors 
use the command without --dru-run flag
```bash
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.org
```

Eventually we just need to make nginx serve our server on 443 
```nginx
server {                                                                                  listen 443 default_server ssl;                                                       listen [::]:443 ssl;

    server_name mangalove.site www.mangalove.site;

    ssl_certificate "/etc/nginx/ssl/live/mangalove.site/fullchain.pem";
    ssl_certificate_key "/etc/nginx/ssl/live/mangalove.site/privkey.pem";

    location / {
        proxy_pass http://backend_mangalove;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
		proxy_set_header Referer $http_referer
    }

    location /static/ {
        root /home/app;
    }

    location /imgs/ {
        root /home/app/;
    }
}
```

Now every request came into 443 ( because every https request come to 443 port)  we just leave the same configuration as before, only added ssl certificate and ssl_certificate_key lines 

Also make sure that your django project has the following setting

```python
CSRF_TRUSTED_ORIGINS = ['https://example.org']
CSRF_COOKIE_HTTPONLY = False
```

***It's done!***


For renewing keys use 
```bash
docker compose run --rm certbot renew
```