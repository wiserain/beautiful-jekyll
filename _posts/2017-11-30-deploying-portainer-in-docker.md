---
layout: post
title: "docker-portainer 설치"
tags: [docker, portainer]
comments: true
share: true
---


### 개요

[portainer](http://portainer.io)는 docker를 관리하기 위한 웹앱이다. Synology DSM에 WebUI로 존재하는 그것과 같은 것이다.

> Portainer is an open-source lightweight management UI which allows you to easily manage your Docker hosts or Swarm clusters

### 설치

간단하게 docker container를 하나 설치하면 내부적으로는 unix socket을 통해서 통신하고 WebUI로 control할 수 있는 환경을 제공한다.

```yaml
version: '2'

services:
  portainer:
    image: portainer/portainer
    container_name: portainer
    restart: always
    network_mode: "bridge"
    volumes:
      - /your/local/portainer/data/path:/data
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "9000:9000"
```

### Reverse Proxy

[공식 문서](https://portainer.readthedocs.io/en/stable/faq.html#how-can-i-configure-my-reverse-proxy-to-serve-portainer)에 Nginx Reverse proxy 설정 예시가 잘 나와있다. 여기서는 [https-portal](https://github.com/SteveLTN/https-portal)과 호환 가능한 nginx.conf를 남긴다.

```conf
server {
	listen 443 ssl http2;
	server_name <%= domain.name %>;

	ssl on;
	ssl_certificate <%= domain.chained_cert_path %>;
	ssl_certificate_key <%= domain.key_path %>;

	ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
	ssl_session_cache shared:SSL:50m;
	ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
	ssl_prefer_server_ciphers on;

	ssl_dhparam <%= dhparam_path %>;

    location / {
        proxy_pass  <%= domain.upstream %>/;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        access_log off;
    }

    location /api/websocket/ {
        proxy_pass  <%= domain.upstream %>/api/websocket/;

		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_http_version 1.1;
        access_log off;
    }
}
```

### 활용

직관적으로 잘 꾸며져 있어서 대부분의 기능을 문제없이 이용할 수 있다. (아직 적극적으로 사용하지 않아서 어떤 제한이 있는지 파악하지 못했다.)

local의 docker service뿐만 아니라 docker에서 제공하는 Remote API를 통해서 외부의 서비스도 연결할 수 있다. 이를 위해서는 target machine의 docker에서 이걸 허용해줘야 하는데, 이 [gist 문서](https://gist.github.com/jupeter/b39e11521452129af2af85cc855c91d7)와 같이 docker 실행 옵션 ```DOCKER_OPTS``` 을 수정해주면 된다.

그런데 버그인지 우분투에서 적용이 안된다. [여기](http://www.littlebigextra.com/how-to-enable-remote-rest-api-on-docker-host/)를 참고해서 ```/lib/systemd/system/docker.service``` 파일을 수정하면 잘 동작한다. docker service 시작 스크립트에 하드코딩으로 집어 넣는 것이다.

그런 다음 ENDPOINT 메뉴에서 IP:PORT를 등록하면 된다.

