---
layout: post
title: Docker 이미지를 이용한 https / let's encrypt / nginx-fpm 통합 구축
tags:
  - docker
  - HTTPS
  - lets encrypt
  - SSL
  - 인증서
comments: true
share: true
from: 'http://wiserain.net/1039'
---

### 목적 및 구성

- 보다 안전한 https 웹서비스
- 자동 갱신/관리 되는 ssl 인증서 with let's encrypt
- 서브 도메인을 이용한 DSM 서비스의 사용, 예) dsm.example.com
- 간단한 웹페이지 운용

### 필요 조건

- 당연히 개인 도메인 (이후는 example.com이라 한다)
- 원하는 서비스에 맞는 원하는 서브 도메인과 개인 서버를 가리키는 네임 호스트 설정
- 도커 패키지가 사용 가능한 환경

### 방법

원리를 간략히 요약하자면 웹서비스에 필요한 80/443 포트의 요청을 docker 컨테이너에 전달하면, ssl 적용 및 subdomain redirect까지 해결해서 docker 호스트로 전달하는 reverse proxy

1\. 포트 포워딩

웹서비스 요청을 전달하기 위해 공유기에서 docker 호스트로 포트포워딩을 한다. 외부 80/443을 임의의 포트인 180/1443로 매핑해 준다.

2\. 컨테이너 생성/실행

아래 docker-compose 설정을 참고하여 컨테이너를 생성한다. [참고](http://wiserain.net/1035)

```yaml
version: '2'

services:
  https:
  container_name: https
  image: steveltn/https-portal
  restart: always
  network_mode: "bridge"
  ports:
    - '180:80'
    - '1443:443'
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - /volume1/docker/https/ssl_certs:/var/lib/https-portal
    - /volume1/docker/https/nginx-conf:/var/lib/nginx-conf:ro
    - /volume1/docker/https/vhosts:/var/www/vhosts
  links:
    - php7fpm
  environment:
    - DOMAINS=
      example.com, rss.example.com,
      www.example.com - > http://dockerhost,
      router.example.com -> http://192.168.1.1:8888,
      dsm.example.com -> http://dockerhost:5000,
      photo.example.com -> http://dockerhost,
      dav.example.com -> http://dockerhost:5005,
      plex.example.com -> http://dockerhost:32400,
      plexpy.example.com -> http://dockerhost:8181,
      flexget.example.com -> http://dockerhost:5050,
      transmission.example.com -> http://dockerhost:9091,
      tvheadend.example.com -> http://dockerhost:9981
    - STAGE=staging
    - FORCE_RENEW=false
    - WORKER_PROCESSES=auto
    - SERVER_NAMES_HASH_MAX_SIZE=8192
    - SERVER_NAMES_HASH_BUCKET_SIZE=64
    - CLIENT_MAX_BODY_SIZE=0

  php7fpm:
    container_name: php7fpm
    image: php:7-fpm
    restart: always
    network_mode: "bridge"
    volumes:
      - /volume1/docker/https/vhosts:/var/www/vhosts
```

메인 컨테이너인 https에서 php를 돌리기 위해 php7fpm 서비스를 링크해서 쓰는 것으로 이해하면 된다.

**사용자 변수 DOMAINS를 통해 서브도메인을 설정** 할 수 있는데 "원하는 도메인 - > 보낼 곳"의 형식으로 쓰면 reverse proxy가 설정된다. plex를 예로 들면,

```yaml
plex.example.com -> http://dockerhost:32400
```

이건 내가 plex를 docker에서 운영하고 있어서 그런 것이고, 다른 적절한 backend를 지정해준다.

위에 적은대로 공유기도 잘되고 DSM도 잘되고 다 잘되는데 **포토스테이션은 기본 설정과는 다른 proxy rule** 을 써야한다.

아래의 내용을 ```photo.example.com.ssl.conf.erb``` 파일로 만들어서 ```/volume1/docker/https/nginx-conf```에 넣어준다.

```ruby
server {
  listen 443 ssl http2;
  server_name  <%= domain.name %>;

  ssl on;
  ssl_certificate <%= domain.chained_cert_path %>;
  ssl_certificate_key <%= domain.key_path %>;

  ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
  ssl_session_cache shared:SSL:50m;
  ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-
  AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-
  AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
  ssl_prefer_server_ciphers on;

  ssl_dhparam <%= dhparam_path %>;

  location / {
    return    301 https://$server_name/photo/;
  }

  location /photo/ {
    proxy_pass <%= domain.upstream %>;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

포토스테이션은 <http://server.com/photo/> 아래에서 돌아가기 때문이다.

DOMAINS 사용자 변수에서 보내는 방향 표시가 없으면 **virtual host로 설정** 한다는 의미이다. 다시 말해 자체적으로 내부의 nginx로 웹서버를 돌리겠다는 의미. 그 내용물이 볼륨 부분에서 설정한 ```/volume1/docker/https/vhosts``` 아래에 저장된다. ```rss.example.com```을 예로 들면,

```
/docker/https/vhosts/rss.example.com/
```

에 저장된다는 얘기다.

여기서 **php를 사용하고 싶다면** 앞에서 포토스테이션의 rule을 바꾼 것처럼 ```default.ssl.conf.erb```를 수정해 줘야한다. 이 파일은 원래 docker 이미지에 있는 것인데, nginx에게 "php 스크립트가 들어오면 link된 php7fpm 서비스로 보내서 처리해와"라고 알려줘야 한다. 아래의 내용을 ```default.ssl.conf.erb``` 파일로 만들어서 ```/volume1/docker/https/nginx-conf```에 넣어준다.

```ruby
server {
  listen 443 ssl http2;
  server_name  <%= domain.name %>;

  ssl on;
  ssl_certificate <%= domain.chained_cert_path %>;
  ssl_certificate_key <%= domain.key_path %>;

  ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
  ssl_session_cache shared:SSL:50m;
  ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-
  AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-
  AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
  ssl_prefer_server_ciphers on;

  ssl_dhparam <%= dhparam_path %>;

  <% if domain.upstream %>
  location / {
    proxy_pass <%= domain.upstream %>;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    <% if ENV['WEBSOCKET'] && ENV['WEBSOCKET'].downcase == 'true' %>
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_read_timeout 2h;
    <% end %>
  }
  <% else %>
  root   <%= domain.www_root %>;
  index  index.php index.html index.htm;

  location / {
    try_files $uri $uri/ /index.php?$query_string;
  }

  # pass the PHP scripts to FastCGI server listening on socket
  #
  location ~ \\.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\\.php)(/.+)$;
    fastcgi_pass php7fpm:9000;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
    fastcgi_index index.php;
    include fastcgi_params;
  }

  location ~* \\.(jpg|jpeg|gif|png|css|js|ico|xml)$ {
    expires           5d;
  }

  # deny access to . files, for security
  #
  location ~ /\\. {
    log_not_found off;
    deny all;
  }

  location ^~ /.well-known {
    allow all;
    auth_basic off;
  }
  <% end %>
}
```

중요한 부분은 저기 44번째 줄 쯤에 ```fastcgi_pass php7fpm:9000;```이다.

그 외 응답할 도메인에 대해 별도의 rule을 만들고 싶으면 ```도메인명.ssl.conf.erb```로 만들어서 올리면 되고(override), 정의되지 않은 rule은 ```default.ssl.conf.erb```를 따른다.

비슷한 방식으로 db까지 연동할 수 있을텐데, 관리의 어려움 때문에 db 들어가는 앱을 운용하지 않는터라 자세한 정보는 잘 모르겠다. [github 소개 페이지](https://github.com/SteveLTN/https-portal) 에 보면 워드프레스를 간단히 연결해서 쓰기도 하니 참고하길 바란다.

기본 nginx.conf의 서버 설정을 그대로 쓰면 처리량이 모자라서 문제가 생기는 경우가 있다. 특히 DSM 같은 경우는 GUI로 구현되어 있고, 그 위로 파일을 막 주고 받기 때문에 에러가 뜨는 경우를 종종 봤다. 이를 대비해서 사용자 변수로 주면 바로 서버 설정에 반영되도록 docker 이미지가 고안되어 있다. docker-compose에서 가장 마지막 부분에 해당하는 이 내용이다.

```yaml
  - WORKER_PROCESSES=auto
  - SERVER_NAMES_HASH_MAX_SIZE=8192
  - SERVER_NAMES_HASH_BUCKET_SIZE=64
  - CLIENT_MAX_BODY_SIZE=0
```

변수화 된 설정 이외에 추가적으로 고치고 싶다면, nginx.conf.erb를 수정해주는 방법이 있다. 고급 사용자는 아마 이쪽이 더 쉬울 것이라 생각한다.

어느 정도 설정을 마쳤으면 이제 실행해 본다. 컨테이너가 생성되고 nginx.conf.erb에 사용자 변수를 대입해 최종적으로 nginx.conf 파일을 만들어서 서버를 세팅하고 별도의 인증서 관련 루틴이 동작하여 지정된 도메인들에 대해 인증서를 발급 받는다. (로그를 보자) 그런데 이대로 실행하면 SSL이 적용되지 않는데, 디폴트로 STAGE=staging이며, 이 값을 production으로 바꿔줘야 let's encrypt 인증을 시작하기 때문이다. 이유가 있는데, 무분별한 인증을 막기 위해서 lets encrypt에서 제한을 두고 있기 때문이다. 구체적으로,

- 3시간에 한 IP당 10번의 registration
- 일주일에 한 도메인 당 5개의 인증서

만 발급을 허용한다. 따라서 등록할 도메인의 양이 많거나, 테스트를 위해서 컨테이너와 볼륨을 지웠다 생성했다 하다보면 금방 막히니 계획적으로 접근해야 한다. 이를 도와주는 것이 STAGE 변수 설정이다.

### 시놀로지 모바일 앱에서는?

위의 설정 예제를 기준으로 dsm.example.com:443으로 꼭 포트를 붙여줘야 한다. 물론 https 설정도. 좀 마음에 안들기는 하지만... 이 모바일 앱들이 기본적으로 5000/5001 포트를 가정하고 있기 때문에 다른 방법이 없다. 포토스테이션 앱 같은 경우에는 그냥 photo.example.com으로 하면 된다.

### 추가 정보

더 자세한 정보는 docker 이미지 제작자 페이지 [SteveLTN](https://github.com/SteveLTN/https-portal) 에서 확인하도록 한다.  또한 유사한 이미지나 그에 대한 설명이 있으니 이 또한 참고하길 바란다. [JrCs](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-
companion), [gilyes](https://gilyes.com/docker-nginx-letsencrypt/)

출처: [https://forum.synology.com/enu/viewtopic.php?t=109439](https://forum.synology.com/enu/viewtopic.php?t=109439https://forum.synology.com/enu/viewtopic.php?t=109439)
