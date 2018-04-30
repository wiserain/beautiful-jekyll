---
layout: post
title: "우분투 16.04 초기 설정"
tags: [ubuntu, 16.04]
comments: true
share: false
---

환경은

- 우분투 16.04 server/desktop


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
from official repository
```bash
sudo apt-get install xrdp
```

or more recent version from ppa
> https://netdevops.me/2017/installing-xrdp-0.9.1-on-ubuntu-16.04-xenial/

```bash
sudo add-apt-repository ppa:hermlnx/xrdp
sudo apt-get update
sudo apt-get install xrdp
```

```bash
sudo sed -i.bak '/fi/a #xrdp multiple users configuration \n mate-session \n' /etc/xrdp/startwm.sh
```

or build the latest by yourself
> http://c-nergy.be/blog/?p=11719

> https://think.unblog.ch/xrdp-remote-desktop-auf-linux/

> https://github.com/neutrinolabs/xrdp/wiki/Building-on-Debian-8

```bash
#---------------------------------------------------#
# Step 1 - Download XRDP Binaries...
#---------------------------------------------------#

mkdir ~/git && cd ~/git
sudo apt-get -y install git

git clone https://github.com/neutrinolabs/xrdp.git
git clone https://github.com/neutrinolabs/xorgxrdp.git

#---------------------------------------------------#
# Step 2 - Install Prereqs...
#---------------------------------------------------#

sudo apt-get -y install libx11-dev libxfixes-dev libssl-dev libpam0g-dev libtool libjpeg-dev flex bison gettext autoconf libxml-parser-perl libfuse-dev xsltproc libxrandr-dev python-libxml2 nasm xserver-xorg-dev fuse

#---------------------------------------------------#
# Step 3 - Check if Fontutil.h file exists...
#---------------------------------------------------#

file="/usr/include/X11/fonts/fontutil.h"

if [ ! -f "$file" ]
then
cat >/usr/include/X11/fonts/fontutil.h <<EOF
#ifndef _FONTUTIL_H_
#define _FONTUTIL_H_

#include <X11/fonts/FSproto.h>

extern int FontCouldBeTerminal(FontInfoPtr);
extern int CheckFSFormat(fsBitmapFormat, fsBitmapFormatMask, int *, int *,
    			 int *, int *, int *);
extern void FontComputeInfoAccelerators(FontInfoPtr);

extern void GetGlyphs ( FontPtr font, unsigned long count,
    			unsigned char *chars, FontEncoding fontEncoding,
    			unsigned long *glyphcount, CharInfoPtr *glyphs );
extern void QueryGlyphExtents ( FontPtr pFont, CharInfoPtr *charinfo,
    				unsigned long count, ExtentInfoRec *info );
extern Bool QueryTextExtents ( FontPtr pFont, unsigned long count,
    			       unsigned char *chars, ExtentInfoRec *info );
extern Bool ParseGlyphCachingMode ( char *str );
extern void InitGlyphCaching ( void );
extern void SetGlyphCachingMode ( int newmode );
extern int add_range ( fsRange *newrange, int *nranges, fsRange **range,
    		       Bool charset_subset );

#endif /* _FONTUTIL_H_ */
EOF

fi

#---------------------------------------------------#
# Step 4 - compiling...
#---------------------------------------------------#

cd ~/git/xrdp
sudo ./bootstrap && \
sudo ./configure --enable-fuse --enable-jpeg && \
sudo make -j4
sudo make install

cd ~/git/xorgxrdp
sudo ./bootstrap && \
sudo ./configure && \
sudo make -j4
sudo make install

#---------------------------------------------------#
# Step 5 - create policies exceptions ....
#---------------------------------------------------#

sudo bash -c "cat >/etc/polkit-1/localauthority.conf.d/02-allow-colord.conf" <<EOF

polkit.addRule(function(action, subject) {
if ((action.id == “org.freedesktop.color-manager.create-device” ||
action.id == “org.freedesktop.color-manager.create-profile” ||
action.id == “org.freedesktop.color-manager.delete-device” ||
action.id == “org.freedesktop.color-manager.delete-profile” ||
action.id == “org.freedesktop.color-manager.modify-device” ||
action.id == “org.freedesktop.color-manager.modify-profile”) &&
subject.isInGroup(“{users}”)) {
return polkit.Result.YES;
}
});
EOF

#---------------------------------------------------#
# Step 6 - configure Xwrapper file  ....
#---------------------------------------------------#

sudo sed -i 's/allowed_users=console/allowed_users=anybody/' /etc/X11/Xwrapper.config

#---------------------------------------------------#
# Step 8 - create services ....
#---------------------------------------------------#

sudo systemctl daemon-reload
sudo systemctl enable xrdp.service
sudo systemctl enable xrdp-sesman.service
sudo systemctl start xrdp

#---------------------------------------------------#
# Step 9 - install additional pacakge ....
#---------------------------------------------------#

sudo apt-get -y install xserver-xorg-core

vmversion=$(sudo dmidecode -s system-product-name)
echo $vmversion
if [ "$vmversion" = "VirtualBox" ]
then
    sudo apt-get -y install xserver-xorg-input-all
fi
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

### error reporting
> https://askubuntu.com/questions/93457/how-do-i-enable-or-disable-apport

Disable
```bash
sudo systemctl disable apport.service
```
If that does not work, you would then need to mask the service
```bash
systemctl mask apport.service
```
To reenable
```bash
systemctl unmask apport.service # if you masked it
sudo systemctl enable apport.service
```

A file editor is now open. Change enabled from "0" to a "1" so it looks like this:
```bash
enabled=1
```
To turn it off make it:
```bash
enabled=0
```
Now save your changes and close the file editor. Apport will now no longer start at boot. If you want to turn it off immediately without rebooting, run ```sudo service apport stop```.

You can also use ```sudo service apport stop``` without modifying ```/etc/default/apport``` to turn it off temporarily.


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
