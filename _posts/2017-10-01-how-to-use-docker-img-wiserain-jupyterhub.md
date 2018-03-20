---
title: wiserain/jupyterhub 이미지 사용법
layout: post
tags: [jupyterhub, docker]
comments: true
---

이전 글과 연계하여 빌드 된 [wiserain/jupyterhub](https://hub.docker.com/r/wiserain/jupyterhub/)를 사용하는 방법

### Network / Volume 생성

**docker network 생성**
```bash
docker network inspect $(DOCKER_NETWORK_NAME) >/dev/null 2>&1 || docker network create $(DOCKER_NETWORK_NAME)
```

**docker volume 생성**
```bash
docker volume inspect $(DATA_VOLUME_HOST) >/dev/null 2>&1 || docker volume create --name $(DATA_VOLUME_HOST)
```

### Github Authentication

인증 처리를 위해 Github OAuth를 이용한다. [여기 설명](https://github.com/jupyterhub/jupyterhub-deploy-docker#authenticator-setup)을 참조해서 진행하고 3가지 정보를 잘 메모한다.

```bash
GITHUB_CLIENT_ID=<github_client_id>
GITHUB_CLIENT_SECRET=<github_client_secret>
OAUTH_CALLBACK_URL=https://<myhost.mydomain>/hub/oauth_callback
```

### 파일 준비

```/volume1/docker/jupyterhub```아래에 다음의 두 파일을 준비한다.

**```jupyterhub_config.py``` 파일 내용**

```py
# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

# Configuration file for JupyterHub
import os

c = get_config()

# We rely on environment variables to configure JupyterHub so that we
# avoid having to rebuild the JupyterHub container every time we change a
# configuration parameter.

# Spawn single-user servers as Docker containers
c.JupyterHub.spawner_class = 'dockerspawner.DockerSpawner'
# Spawn containers from this image
c.DockerSpawner.container_image = os.environ['DOCKER_NOTEBOOK_IMAGE']
# JupyterHub requires a single-user instance of the Notebook server, so we
# default to using the `start-singleuser.sh` script included in the
# jupyter/docker-stacks *-notebook images as the Docker run command when
# spawning containers.  Optionally, you can override the Docker run command
# using the DOCKER_SPAWN_CMD environment variable.
spawn_cmd = os.environ.get('DOCKER_SPAWN_CMD', "start-singleuser.sh")
c.DockerSpawner.extra_create_kwargs.update({ 'command': spawn_cmd })
# Connect containers to this Docker network
network_name = os.environ['DOCKER_NETWORK_NAME']
c.DockerSpawner.use_internal_ip = True
c.DockerSpawner.network_name = network_name
# Pass the network name as argument to spawned containers
c.DockerSpawner.extra_host_config = { 'network_mode': network_name }
# Explicitly set notebook directory because we'll be mounting a host volume to
# it.  Most jupyter/docker-stacks *-notebook images run the Notebook server as
# user `jovyan`, and set the notebook directory to `/home/jovyan/work`.
# We follow the same convention.
notebook_dir = os.environ.get('DOCKER_NOTEBOOK_DIR') or '/home/jovyan/work'
c.DockerSpawner.notebook_dir = notebook_dir
# Mount the real user's Docker volume on the host to the notebook user's
# notebook directory in the container
c.DockerSpawner.volumes = { 'jupyterhub-user-{username}': notebook_dir }
c.DockerSpawner.extra_create_kwargs.update({ 'volume_driver': 'local' })
# Remove containers once they are stopped
c.DockerSpawner.remove_containers = True
# For debugging arguments passed to spawned containers
c.DockerSpawner.debug = True

# User containers will access hub by container name on the Docker network
c.JupyterHub.hub_ip = 'jupyterhub'
c.JupyterHub.hub_port = 8080

# TLS config
# c.JupyterHub.port = 443
# c.JupyterHub.ssl_key = os.environ['SSL_KEY']
# c.JupyterHub.ssl_cert = os.environ['SSL_CERT']

# Authenticate users with GitHub OAuth
c.JupyterHub.authenticator_class = 'oauthenticator.GitHubOAuthenticator'
c.GitHubOAuthenticator.oauth_callback_url = os.environ['OAUTH_CALLBACK_URL']

# Persist hub data on volume mounted inside container
data_dir = os.environ.get('DATA_VOLUME_CONTAINER', '/data')
c.JupyterHub.db_url = os.path.join('sqlite:///', data_dir, 'jupyterhub.sqlite')
c.JupyterHub.cookie_secret_file = os.path.join(data_dir,
    'jupyterhub_cookie_secret')

# Whitlelist users and admins
c.Authenticator.whitelist = whitelist = set()
c.Authenticator.admin_users = admin = set()
c.JupyterHub.admin_access = True
pwd = os.path.dirname(__file__)
with open(os.path.join(pwd, 'userlist')) as f:
    for line in f:
        if not line:
            continue
        parts = line.split()
        name = parts[0]
        whitelist.add(name)
        if len(parts) > 1 and parts[1] == 'admin':
            admin.add(name)
```

```docker-compose.yml``` 파일 내용
```
user1 admin
user2
user3
```

### docker-compose로 실행

```yaml
version: '2'

services:
  jupyterhub:
    image: wiserain/jupyterhub:latest
    container_name: jupyterhub
    restart: always
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:rw"
      - "data:/data"
      - "/volume1/docker/jupyterhub:/srv/jupyterhub"
    ports:
      - "8000:8000"
    environment:
      DOCKER_NETWORK_NAME: jupyterhub-network
      DOCKER_NOTEBOOK_IMAGE: jupyter/scipy-notebook:latest
      DOCKER_NOTEBOOK_DIR: /home/jovyan/work
      DOCKER_SPAWN_CMD: start-singleuser.sh
      GITHUB_CLIENT_ID: <github_client_id>
      GITHUB_CLIENT_SECRET: <github_client_secret>
      OAUTH_CALLBACK_URL: https://<myhost.mydomain>/hub/oauth_callback

volumes:
  data:
    external:
      name: jupyterhub-data

networks:
  default:
    external:
      name: jupyterhub-network
```


### nginx reverse proxy
이제 localhost:8000으로 서비스가 된다. 아래의 nginx.conf 설정을 추가하여 도메인과 연결해 준다. WebSocket을 열어주는 것이 관건.

```ruby
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
        proxy_pass <%= domain.upstream %>;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

		proxy_set_header X-NginX-Proxy true;
    }

	location ~* /(api/kernels/[^/]+/(channels|iopub|shell|stdin)|terminals/websocket)/? {
		proxy_pass <%= domain.upstream %>;

		proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

		proxy_set_header X-NginX-Proxy true;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
		proxy_read_timeout 86400;
	}
}
```
