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


**20180406 추가**

```docker ps```와 달리 ```docker stats```는 docker container에 대한 CPU 사용률, 메모리 사용량, Network IO를 보여주는 명령어인데 아래와 같이 컨테이너 이름이 SHA1로 나와서 알아보기 어렵게 되어 있다.

```bash
CONTAINER           CPU %               MEM USAGE / LIMIT       MEM %               NET I/O             BLOCK I/O           PIDS
1f13cd899e63        0.31%               520.2 MiB / 7.796 GiB   6.52%               2.86 GB / 33.3 MB   16.8 GB / 636 MB    38
5e5fb3f0166b        0.26%               163.5 MiB / 7.796 GiB   2.05%               4.27 GB / 3 GB      388 MB / 68.8 MB    38
44d0bdc0b730        0.02%               33.44 MiB / 7.796 GiB   0.42%               0 B / 0 B           127 MB / 29.8 MB    31
7002a1492b11        0.25%               43.64 MiB / 7.796 GiB   0.55%               0 B / 0 B           121 MB / 5.83 GB    30
2b70edaba352        37.04%              14.39 MiB / 7.796 GiB   0.18%               26.7 MB / 2.93 MB   58.1 MB / 7.66 GB   10
1fb1c6babf88        0.00%               5.297 MiB / 7.796 GiB   0.07%               24 MB / 5.62 MB     63.4 MB / 90.1 kB   10
c33bd07cab2c        0.11%               77.56 MiB / 7.796 GiB   0.97%               76.1 MB / 20.6 MB   386 MB / 264 MB     41
eab5d1f7189b        0.26%               380.9 MiB / 7.796 GiB   4.77%               0 B / 0 B           3.05 GB / 4.8 GB    78
7c86675bceda        0.65%               17.36 MiB / 7.796 GiB   0.22%               1.39 GB / 47.9 MB   480 MB / 590 kB     26
9c7510f0131d        0.00%               20.75 MiB / 7.796 GiB   0.26%               2.98 GB / 2.69 GB   361 MB / 32.6 MB    4
390ea85dbb1e        1.18%               324.4 MiB / 7.796 GiB   4.06%               134 MB / 91.2 MB    7.95 GB / 295 kB    43
55ce49ce6948        0.00%               2.128 GiB / 7.796 GiB   27.29%              677 MB / 244 MB     3.36 GB / 1.38 GB   18
af91bea81b3b        0.03%               37.11 MiB / 7.796 GiB   0.46%               47.3 MB / 990 kB    389 MB / 52.4 MB    31
9c9670292c5f        3.15%               28.92 MiB / 7.796 GiB   0.36%               80.3 MB / 60.3 MB   439 MB / 81.9 kB    10
```

아래 명령어는 container name을 human readable하게 고쳐준다.

```bash
docker stats $(docker ps --format={{.Names}})
```
