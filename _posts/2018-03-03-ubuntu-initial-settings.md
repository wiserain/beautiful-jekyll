---
layout: post
title: "우분투 초기 설정"
tags: [ubuntu, 16.04]
comments: true
share: false
---

환경은

- 우분투 16.04 server


### 다음 카카오로 저장소 변경
- https://openwiki.kr/tech/ubuntu_daumkakao_repository
- http://bloodygale.tistory.com/entry/Ubuntu-%EC%A0%80%EC%9E%A5%EC%86%8C-%EB%B3%80%EA%B2%BD

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i 's/kr.archive.ubuntu.com/ftp.daumkakao.com/g' /etc/apt/sources.list
sudo apt-get update
sudo apt-get upgrade
```

### locale 확인
```/etc/default/locale``` 파일을 찾아 다음과 같이 수정. (각 행의 마지막에 빈칸이 포함되지 않도록 한다.)

```bash
LANG="en_US.UTF-8"
LANGUAGE="en_US.UTF-8"
LC_ALL="en_US.UTF-8"
```

### mate-desktop
```bash
sudo apt-add-repository ppa:ubuntu-mate-dev/xenial-mate # currently 1.16
```
or

```bash
sudo add-apt-repository ppa:jonathonf/mate-1.18 # if you want 1.18
sudo apt update
sudo apt-get install mate-core mate-desktop-environment mate-notification-daemon caja-open-terminal ubuntu-mate-icon-themes -y
```

### xrdp
```bash
sudo apt-get install xrdp
```
or
> https://netdevops.me/2017/installing-xrdp-0.9.1-on-ubuntu-16.04-xenial/

```bash
sudo add-apt-repository ppa:hermlnx/xrdp
sudo apt-get update
sudo apt-get install xrdp
```

```bash
sudo sed -i.bak '/fi/a #xrdp multiple users configuration \n mate-session \n' /etc/xrdp/startwm.sh
```

### sublimetext 3
```bash
wget -qO - https://download.sublimetext.com/sublimehq-pub.gpg | sudo apt-key add -
sudo apt-get install apt-transport-https
```

stable

```bash
echo "deb https://download.sublimetext.com/ apt/stable/" | sudo tee /etc/apt/sources.list.d/sublime-text.list
```

install

```bash
sudo apt-get update
sudo apt-get install sublime-text
```

### misc
기타 프로그램

```bash
sudo apt-get install landscape-common intel-gpu-tools htop firefox docker-compose
```

cifs/smb 마운트

```bash
sudo mount -t cifs -o username=계정이름,password=암호,uid=자신의uid,gid=자신의gid,iocharset=utf8,codepage=cp949 //컴퓨터이름(혹은 주소)/공유이름 /공유할/디렉토리
```

fstab 마운트

crontab 설정

### 한국어 설정

desktop 환경을 가정하면

```bash
sudo apt-get install language-pack-ko language-pack-ko-base
```
그런데 위는 일반적인 unity desktop 기반인 것 같고, gnome 기반의 mate desktop은 다른 패키지가 필요하다. 그냥 language support를 통해서 하는걸 추천

language support가 없다면?
```bash
sudo apt-get install language-selector-gnome
```

시스템 설정 >> language support에서 한글 언어팩을 설치한다. 알아서 기본 폰트와 firefox 언어팩도 설치해 준다.

그런 다음 한글 입력기 설치

```bash
sudo apt-get install fcitx-hangul
```

추가로 고려할 만한 한글 폰트

```bash
sudo apt-get install fonts-nanum* fonts-noto-cjk fonts-unfonts
```
