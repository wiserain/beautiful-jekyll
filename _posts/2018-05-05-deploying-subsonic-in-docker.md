---
layout: post
title: "docker-subsonic 설치"
tags: [docker, subsonic]
comments: true
share: false
---


### 개요

[subsonic](http://www.subsonic.org)은 설치형 음악 스트림 서비스이다. MelOn처럼 웹 접속으로 들을 수도 있고, API가 공개되어 있어 모바일 앱 지원도 풍부하고, Linux/MAC/Windows 플랫폼 지원도 좋은 편이라 일찍부터 넓은 인지도와 두터운 사용자 층을 확보하고 있다.

약간의 역사가 있는데 [6.0-beta1 2016년 3월](https://sourceforge.net/projects/subsonic/files/subsonic/) 이후로 개발자가 오픈소스 중단을 선언하였다. 여기에 반하여 누군가 [Libresonic](https://github.com/Libresonic/libresonic)을 만들었으나 maintainer인 Eugene E. Kashpureff Jr.에게 대다수의 contributor와는 다른 의도와 목적을 가지고 있음이 드러나 또다시 fork/rebrand를 결정하여 [airsnonic](https://github.com/airsonic/airsonic)이 탄생하였다.

Libresonic은 2017년 이후로 프로젝트가 거의 죽은 것 같고, airsonic은 아직 활발하게 진행 중이다. 며칠 전에도 [commit](https://github.com/airsonic/airsonic/commits/master)이 있다.

여기서는 subsonic에 대해 적는다. 같은 API를 따르고 있는만큼 설치와 사용에 있어서 airsonic도 별반 다르지 않을 것이라고 생각한다.


### 설치

여러 이미지를 설치해봤는데 [hurricane/subsonic](https://hub.docker.com/r/hurricane/subsonic/)이 가장 무난했다.

```yaml
version: '2'

services:
  subsonic:
    image: hurricane/subsonic
    container_name: subsonic
    restart: always
    network_mode: "bridge"
    volumes:
      - ${DOCKER_ROOT}/subsonic/config:/subsonic
      - ${DOCKER_ROOT}/subsonic/music:/music
      - ${DOCKER_ROOT}/subsonic/podcast:/podcasts
      - ${DOCKER_ROOT}/subsonic/playlists:/playlists
      - /volume1/music:/volume1/music
    ports:
      - "4040:4040"
    environment:
      - TZ=Asia/Seoul
      - APP_USER=chulwoo
      - APP_UID=1000
      - APP_GID=1000
```

### Reverse Proxy

특별한 다른 설정 없이 그냥 proxy_pass로 넘겨주면 된다. 여기서는 [https-portal](https://github.com/SteveLTN/https-portal)과 호환 가능한 nginx.conf를 남긴다.

```conf
server {
    listen 443 ssl http2;
    server_name <%= domain.name %>;

    ssl on;
    ssl_certificate <%= domain.signed_cert_path %>;
    ssl_certificate_key <%= domain.key_path %>;

    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:50m;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_prefer_server_ciphers on;

    ssl_dhparam <%= dhparam_path %>;

    # Prevent Nginx from leaking the first TLS config
    if ($host != $server_name) {
        return 444;
    }

    location / {
        proxy_pass  <%= domain.upstream %>;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect http:// https://;
    }
}
```
