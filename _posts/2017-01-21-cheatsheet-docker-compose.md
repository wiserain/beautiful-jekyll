---
layout: post
title: docker-compose 사용법 및 예제
tags: [docker, docker-compose]
comments: true
share: true
from: 'http://wiserain.net/1035'
---

보통 커맨드 라인 명령어인 [docker run](https://docs.docker.com/engine/reference/commandline/run/)을 이용해서 container를 생성/실행한다. plex로 예를 들면,

```bash
docker run -d \
  --name=<container name> \
  --net=host \
  --restart=always \
  -p 32400:32400 \
  -v /video:/video \
  -e VERSION=latest
  <image name> <additional command>
```

실행 후에 멈추고자 하면,

```bash
docker stop <container name or ID>
```

지우려면,

```bash
docker rm <container name or ID>
```

이미지까지 지우려면,

```bash
docker rmi <image name or ID>
```

docker-compose를 사용하면 조금 더 정돈된 규칙으로 컨테이너를 정의할 수 있고 이걸 통해서 대부분의 컨트롤을 할 수 있다. 짐작컨대 docker을 연결해주는 파이썬 기반의 스크립트 인터페이스라고 하면 좋을 것 같다.

먼저 ```docker-compose.yml``` 파일을 에디터로 만든다. 예제부터,

```yaml
version: '2'
  services:
    plex:
      container_name: plex
      image: linuxserver/plex:latest
      restart: always
      network_mode: "host"
      ports:
        - '32400:32400'
      volumes:
        - /volume1/video:/volume1/video
        - /volume1/docker/plex/config:/config
        - /volume1/docker/plex/transcode:/transcode
      environment:
        - PUID=0
        - PGID=0
        - TZ=Asia/Seoul
        - VERSION=latest

    plexpy:
      ...
      ...
```

yml 문법은 아는 사람은 알겠지만 띄어쓰기 두번으로 다른 카테고리를 표현하니 참고. 가끔 indent를 맞추기 위해서 에디터에서 자동 탭이 먹히는 경우가 있으니 유의.

활발하게 개발되고 있는 중이라 version이 1부터 2, 2.3, 3 등등 많고 거기에 따라 문법이 조금 다른데 version 2가 가장 널리 쓰이는 것 같다. 마치 파이썬 같은...

각각의 서비스를 명명하고 컨테이너 이름 각종 옵션 등을 yml 문법에 맞게 써주면 된다.

이제 생성/실행(docker run에 대응하는 명령어)하려면

```bash
docker-compose up -d <service name, e.g. plex>
```

뒤에 서비스 이름이 지정되지 않으면 몽땅 다 적용된다. -d 옵션은 백그라운드로 보내라는 것. demonize의 약자인 듯. 이미 실행되어 있을 때 또 up을 하면 기반 이미지가 달라졌을 때 자동으로 업데이트도 된다.

멈추고 싶으면,

```bash
docker-compose stop <service name>
```

마치 컴퓨터에 전원 버튼을 눌러서 정상적으로 끄는 것처럼 프로세스를 하나하나 종료하고 끈다. 반대로 그냥 전원을 내리려면, kill을 쓴다.

지우려면,

```bash
docker-compose rm <service name>
```

뭐 이정도면 알면 되겠다.
