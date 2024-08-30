# https 적용하기

## nginx, certbot, certbot nginx 플러그인 설치
```
sudo apt-get install nginx

sudo apt update
sudo apt-get install certbot python3-certbot-nginx -y

certbot --version
```

## 먼저 http 해당 서비스에 적용
```
cd /etc/nginx/sites-available/
sudo vi todayfin.site
```

```
server {
    listen 80;
    server_name todayfin.site;

    location / {
        proxy_pass http://192.168.1.94:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
- localhost로 하면 안됨!!

## 기존 default 링크 삭제, 새 링크 추가
```
sudo rm ../sites-enabled/default
sudo ln -s /etc/nginx/sites-available/todayfin.site /etc/nginx/sites-enabled/
sudo nginx -t  #검사
sudo systemctl restart nginx
```


## certbot 으로 인증서 발급
```
sudo certbot --nginx -d todayfin.site -d www.todayfin.site -d front.todayfin.site

sudo vi todayfin.site
```

```
server {
    listen 80;
    server_name todayfin.site;

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS 설정: covyshin.kr 에 대한 처리
server {
    listen 443 ssl;
    server_name todayfin.site;

    ssl_certificate /etc/letsencrypt/live/todayfin.site/fullchain.pem; 
    ssl_certificate_key /etc/letsencrypt/live/todayfin.site/privkey.pem; 
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://192.168.1.94:3000; # 프록시 목표 서버
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
```
sudo nginx -t
sudo systemctl restart nginx
```