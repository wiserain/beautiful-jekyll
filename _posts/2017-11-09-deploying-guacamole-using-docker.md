---
layout: post
title: "docker-guacamole 설치"
tags: [docker, guacamole]
comments: true
share: true
---


## 개요

[guacamole](https://guacamole.incubator.apache.org/)은 소개 페이지에도 잘 나와 있지만 특정 클라이언트 프로그램 없이 웹브라우저로 원격 접속을 가능하게 해주는 웹앱이다. vnc, ssh, rdp를 지원하고, http 위에 돌아가니 포트가 막혀있는 상황에서도 접속이 가능하다.

docker-guacamole은 3가지 이미지로 구현 되는데, 먼저 demon이 있고 tomcat위에 돌아가는 frontend가 있고 mysql이나 postgresql 같은 db가 삼위일체로 돌아간다. 이걸 사용하게 쉽게 만들어 놓은 (mysql이 통합되어 있음) zuhkov/guacamole 이미지가 있지만, 빌드가 2년도 더 되었고 에러도 좀 있는 것 같고 완전하지 않은 것 같아 공식 이미지를 이용한 방법을 간략하게 서술한다. 혹시 zuhkov/guacamole 이미지를 이용한 설치가 궁금하다면 [여기](http://isulnara.com/wp/archives/1166)를 참고한다.

공식 매뉴얼을 포함한 몇 가지 인터넷 문서를 기반으로 설명한다.
* <https://guacamole.incubator.apache.org/doc/gug/guacamole-docker.html#guacamole-docker-guacd>
* <https://www.cb-net.co.uk/linux/running-guacamole-from-a-docker-container-on-ubuntu-16-04-lts-16-10/>
* <https://gist.github.com/kernel-sanders/a900973e0992cfdf4178> (esxi console 접속하는 방법 포함)


## 설치하기

먼저 이미지를 다운로드하고, mysql db의 초기 설정을 위한 작업을 한다. db에는 사용자 계정, 접속할 서버 정보 등이 저장된다.

```bash
# reference
# https://www.cb-net.co.uk/linux/running-guacamole-from-a-docker-container-on-ubuntu-16-04-lts-16-10/

# Pull the guacamole (and related) docker images
sudo docker pull guacamole/guacd
sudo docker pull guacamole/guacamole
sudo docker pull mysql

# Create script to prepare MySQL Database
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql

# Make a script folder to pass-through to container
mkdir -p /volume1/docker/guacamole/scripts
mv initdb.sql /volume1/docker/guacamole/scripts
```

docker-compose를 이용해서 아래와 같이 설정하고 실행한다.

```yaml
# create and run using docker-compose
  guac-mysql:
    image: mysql:latest
    container_name: guac-mysql
    volumes:
      - "/volume1/docker/guacamole/mysql:/var/lib/mysql"
      - "/volume1/docker/guacamole/scripts:/tmp/scripts"
    environment:
      MYSQL_ROOT_PASSWORD: <MYSQL_ROOT_PASSWORD>
```

```bash
docker-compose up -d guac-mysql
```

guac-mysql 컨테이너 안으로 들어가서 db를 생성하고 사용자 비번 등을 설정한다.

```bash
# going into the container
docker-compose exec guac-mysql bash

# Create mysql db, user and prepare mysql instance for guacamole
# https://guacamole.incubator.apache.org/doc/gug/jdbc-auth.html#jdbc-auth-mysql
mysql -u root -p'<MYSQL_ROOT_PASSWORD>'
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user' IDENTIFIED BY '<MYSQL_PASSWORD>';
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO 'guacamole_user';
FLUSH PRIVILEGES;
quit

cat /tmp/scripts/initdb.sql | mysql -u root -p guacamole_db
Enter password: <MYSQL_ROOT_PASSWORD>
```

docker guac-daemon을 정의하고 실행한다.

```yaml
# create and run guacd
  guacd:
    image: guacamole/guacd
    container_name: guacd
    network_mode: "bridge"
```

```bash
docker-compose up -d guacd
```

이제 메인 컨테이너를 지정하고 실행한다.

```yaml
# create and run guacamole main container
  guac:
    image: guacamole/guacamole
    container_name: guacamole
    restart: always
    network_mode: "bridge"
    volumes:
      - /volume1/docker/guacamole/config:/var/lib/guacamole
    links:
      - "guacd:guacd"
      - "guac-mysql:mysql"
    ports:
      - "8080:8080"
    environment:
      MYSQL_DATABASE: guacamole_db
      MYSQL_USER: guacamole_user
      MYSQL_PASSWORD: <MYSQL_PASSWORD>
    depends_on:
      - guac-mysql
      - guacd
```

```bash
docker-compose up -d guacamole

# sometimes the container starts up slowly. haveged can be a remedy.
# https://www.digitalocean.com/community/tutorials/how-to-setup-additional-entropy-for-cloud-servers-using-haveged
```

## 접속하기

이제 ```http://<docker host IP>:8080/guacamole/```로 접속하면 되고 기본 username/password는 guacadmin/guacadmin이다.


