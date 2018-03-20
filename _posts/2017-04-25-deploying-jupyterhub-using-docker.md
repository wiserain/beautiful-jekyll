---
layout: post
title: "docker-jupyterhub 설치"
tags: [docker, jupyterhub]
comments: true
share: true
---

## 소개

jupyterhub는 단일 사용자를 위한 jupyter notebook을 다중 사용자 환경으로 확장한 개념이다. jupyter notebook은 구 ipython notebook이 발전하여 새로운 이름을 가지게 된 것인데, 웹 기반 인터렉티브 IDE라고 보면 된다. Python이 기본이지만 [추가 커널](https://github.com/jupyter/jupyter/wiki/Jupyter-kernels) 설치를 통해 다른 언어도 사용 가능하며 입력한 코드를 바로바로 실행해서 결과를 확인해볼 수 있다.

잘 꾸미면 웹에서 언제 어디서나 접근 가능한 나만의 MATLAB이 하나 생기는 것이다. 요새 MATLAB 라이센스 때문에 대세인 파이썬을 사용하려는 움직임이 있으니 배워두는 것도 나쁘진 않다. (라고 말하기엔 이미 몇년 되었다.)

## 우분투에서 설치하기

리눅스의 레퍼런스라 할 수 있는 우분투에서는 설치가 비교적 간단하다.

구글링 하면 정보가 몇가지 있는데...

* <https://dobest.io/gettings-started-with-jupyterhub/>
* <http://haanjack.github.io/jupyter/ipython/2016/03/08/jupyter.html>
* <http://enginius.tistory.com/628>
* <http://goodtogreate.tistory.com/entry/IPython-Notebook-%EC%84%A4%EC%B9%98%EB%B0%A9%EB%B2%95>

아무래도

* [공식 Github Repository의 설명](https://github.com/jupyterhub/jupyterhub)
* <http://yhzion.tistory.com/17>

이 가장 직관적이고 쉬웠다.

간략한 설치 과정은 기반이 되는 python3를 깔고, nodejs와 configurable-http-proxy를 설치한 다음 notebook을 업그레이드하면 된다.

그냥 그대로 실행하면 기본 설정이 적용되고, 아래 명령어를 치면 설명이 포함된 ```jupyterhub_config.py``` 파일이 생성된다.

```bash
jupyterhub --generate-config
```

이걸 입맛대로 수정해서 적용하면 된다.

시스템 서비스로 등록하려면 다음 링크를 참고한다.

* <https://github.com/jupyterhub/jupyterhub/wiki/Run-jupyterhub-as-a-system-service>
* <http://ezcocoa.com/?p=2376>


## docker-jupyterhub 설치하기
docker를 사용하면 좀 더 체계적으로 설치할 수 있는데, 이미 친절한 예제가 있다. [README](https://github.com/jupyterhub/jupyterhub-deploy-docker#jupyterhub-deploy-docker)를 보면 작동 방식을 그림으로 보여주고 있는데, main container에서 jupyterhub가 돌아가면서 사용자의 요청을 받아 인증하고 각 사용자에게 독립된 notebook container를 spawner를 통해서 자동으로 생성/할당한다.

지금부터의 설명은 github의 README를 번역한 정도에 불과하니 애매모호한 사항이 있으면 직접 가서 보는 편을 추천한다.

### Github Authentication

인증 처리를 위해 Github OAuth를 이용한다. [여기 설명](https://github.com/jupyterhub/jupyterhub-deploy-docker#setup-github-authentication)을 참조해서 진행하고 3가지 정보를 잘 메모한다.

```bash
GITHUB_CLIENT_ID=<github_client_id>
GITHUB_CLIENT_SECRET=<github_client_secret>
OAUTH_CALLBACK_URL=https://<myhost.mydomain>/hub/oauth_callback
```

### Jupyterhub docker image 빌드하기

인증서나 개인 설정이 많아서 모든 사람의 입맛에 맞는 범용 docker-image는 사실상 어렵다. 결국은 스스로 파일 추가해서 build를 해야하는데 [설명](https://github.com/jupyterhub/jupyterhub-deploy-docker#build-the-jupyterhub-docker-image)에 따르면

먼저 https 인증서를 ```/secrets``` 폴더에 넣는다. 나는 별도의 https 요청을 처리하는 nginx reverse proxy를 쓰는 관계로 이 과정은 건너 뛰었다. 다음은 작업하는 root 폴더(```Dockerfile.jupyterhub```와 같은 위치)에 ```userlist``` 파일을 생성한다. 한줄에 github username을 적고 admin 권한을 가질 user만 한줄 띄우고 admin을 붙여준다.

그런 뒤 ```make build```를 하면 이미지가 만들어지는데, make가 깔려 있지 않아 그냥 ```Makefile```안의 내용을 수동으로 했다.

docker network가 없으면 만들고,

```bash
docker network inspect $(DOCKER_NETWORK_NAME) >/dev/null 2>&1 || docker network create $(DOCKER_NETWORK_NAME)
```

docker volume도 만들어 준다.

```bash
docker volume inspect $(DATA_VOLUME_HOST) >/dev/null 2>&1 || docker volume create --name $(DATA_VOLUME_HOST)
```

```Makefile```을 보면 알겠지만 ```$(DOCKER_NETWORK_NAME)```이나 ```$(DATA_VOLUME_HOST)```는 ```.env``` 파일에서 불러오는데 나는 그냥 이것도 수동으로 해 주었다. 한 번만 하면 되니까...

```jupyterhub_config.py```도 필요한 파일이니 준비한다. 나 같이 별도의 reverse proxy를 쓰는 경우라면 아래 3개 항목을 주석처리하면 되고 아니라면 고칠 것이 없다.

```bash
c.JupyterHub.port
c.JupyterHub.ssl_key
c.JupyterHub.ssl_cert
```

```docker-compose.yml```에 서비스도 추가하고, (난 이미 기존에 사용하던 파일이 있어서 붙여넣기 했다.)

```yml
version: "2"

services:
  jupyterhub:
    build:
      context: jupyterhub
      dockerfile: Dockerfile.jupyterhub
    image: jupyterhub
    container_name: jupyterhub
    volumes:
      # Bind Docker socket on the host so we can connect to the daemon from within the container
      - "/var/run/docker.sock:/var/run/docker.sock:rw"
      # Bind Docker volume on host for JupyterHub database and cookie secrets
      - "data:/data"
    ports:
      - "8000:8000"
    environment:
      # All containers will join this network
      DOCKER_NETWORK_NAME: jupyterhub-network
      # JupyterHub will spawn this Notebook image for users
      DOCKER_NOTEBOOK_IMAGE: jupyter/scipy-notebook:latest
      # Notebook directory inside user image
      DOCKER_NOTEBOOK_DIR: /home/jovyan/work
      # Using this run command (optional)
      DOCKER_SPAWN_CMD: start-singleuser.sh
      # Required to authenticate users using GitHub OAuth
      GITHUB_CLIENT_ID: <github_client_id>
      GITHUB_CLIENT_SECRET: <github_client_secret>
      OAUTH_CALLBACK_URL: https://<myhost.mydomain>/hub/oauth_callback
    command: >
      jupyterhub -f /srv/jupyterhub/jupyterhub_config.py

volumes:
  data:
    external:
      name: jupyterhub-data

networks:
  default:
    external:
      name: jupyterhub-network
```

여기서 내가 수정한 것은,

* service name ```hub >> jupyterhub``` (내가 원하는 이름으로)
* context ```. >> jupyberhub``` (```docker-compose.yml```과 ```Dockerfile.jupyterhub```파일이 서로 다른 경로에 존재하기 때문에)
* ports ```443 >>> 8000``` (나는 별도 proxy를 쓰기에)
* 그리고 모든 환경 변수들은 ```.env```에서 참고하면 된다.

이제 필요한 파일이 모두 준비되었을 것이다.

```bash
docker-compose.yml
jupyterhub/secrets/jupyterhub.crt
jupyterhub/secrets/jupyterhub.key
jupyterhub/userlist
jupyterhub/jupyterhub_config.py
jupyterhub/Dockerfile.jupyterhub
```

환경변수도 다 제대로 입력이 되었다면 이제 빌드하면 된다.

```bash
docker-compose build <service name, e.g. jupyterhub>
```

### notebook image 선택

notebook image도 이미 체계적(stack)으로 만들어져 있다. [참고](https://github.com/jupyter/docker-stacks) 위에 ```docker-compose.yml```을 보면 알겠지만, 나는 ```jupyter/scipy-notebook:latest```을 선택했다. spark 같은 수치해석은 필요없고, tensoflow까지면 충분한데 지금 당장은 tensorflow를 쓰지 않아서... ```scipy-notebook```만해도 이미지 용량이 5기가이니 참고.

빌드시에는 spawn을 하지 않으니 상관 없지만 jupyterhub를 실행해서 개인  서버를 실행하기 전에는 이미지가 다운로드 되어 있어야 한다.
```
docker pull jupyter/scipy-notebook:latest
```

### nginx reverse proxy와 연계

별도의 nginx reverse proxy를 통할 경우 아래를 참고해서 ```.conf```파일을 설정한다.

* [Example with nginx reverse proxy](http://jupyterhub.readthedocs.io/en/latest/config-examples.html#example-with-nginx-reverse-proxy)
* [zonca/nginx.conf](https://gist.github.com/zonca/08c413a37401bdc9d2a7f65a7af44462#file-nginx-conf-L48)
