---
layout: post
title: 시놀로지 SSH 비밀번호 없이 rsa key로 접속하기
tags:
  - putty
  - ssh
  - ssh-rsa
  - synology
comments: true
share: true
from: 'http://wiserain.net/1040'
---

비밀번호 기반이 아니라 인증키 기반으로 접속하는 방법

환경은

- 시놀로지 6.0.2
- Windows PuTTY 0.68

1\. 먼저 disable되어 있는 sshd의 설정을 변경해준다. ssh 루트로 접속해서

```
vi /etc/ssh/sshd_config
```

50번째 그리고 54번째 줄의 항목 2개를 주석 제거하여 고친다.

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20170401_001.png)

적용하려면 아래 명령어를 치면 된다는데 뭔가 꼬인 것인지 에러가 나서 그냥 DSM 웹에서 껐다가 켰다.

```bash
sudo synoservicectl --restart sshd
```

2\. ssh-rsa key pair를 만든다. PuTTY에서 쓸 것이니 PuTTY에서 만드는 것이 낫다. PuTTY와 딸려오는 PUTTYGEN이란 프로그램을 쓰면 된다. 버전은 0.68 이상으로 쓰는 것을 권장한다. 0.67과 인터페이스가 달라서 삽질했다.

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20170401_002.png)

RSA로 놓고 만들면 된다. 빈 칸에 마우스를 움직여 달라니 움직여 준다. 안 움직이면 진행바가 안올라감.

우리가 필요한 것은 서버에 심을 공개키와 PuTTY 프로그램에서 읽어서 접속 시 인증에 사용할 개인키.

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20170401_003.png)

개인키는 Save private key 버튼으로 하면 되고, 공개키는 텍스트 창에 나와있는 ssh-rsa로 시작하는 내용을 복사 붙여넣기 할 것이니 창을 끄지 말고 대기할 것.

3\. 공개키를 서버에 넣어주자. 만약 abc라는 유저명으로 접속한다면 공개키를 읽어오는 경로가 ```/var/services/homes/abc/.ssh/authorized_keys```이다. 따라서 사용자 개인홈 활성화가 선행되어야 한다. 위에 ssh-rsa로 시작하는 내용을 붙여넣자. 한 200자리쯤 되는 문자열.

```bash
vi /var/services/homes/abc/.ssh/authorized_keys
```

키가 맞는지 체크해볼 수도 있다.

```bash
ssh-keygen -l -f /var/services/homes/abc/.ssh/authorized_keys
```

권한이 중요한데 아래와 같이 맞춰주면 된다.

```bash
chmod 755 /var/services/homes/abc
chmod 700 /var/services/homes/abc/.ssh
chmod 600 /var/services/homes/abc/.ssh/authorized_keys
```

4\. PuTTY로 접속

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20170401_004.png)

아까 저장해둔 개인키를 로드하고,

![]({{ site.url }}{{ site.baseurl }}/img/PicPick_Capture_20170401_005.png)

유저명도 입력하기 싫으면 여기에 등록해준 다음 세션을 저장하고 로그인하면 된다.


참고한 글
- <https://forum.synology.com/enu/viewtopic.php?t=126166>
- <https://www.chainsawonatireswing.com/2012/01/15/ssh-into-your-synology-diskstation-with-ssh-keys/>
- <http://thisguyknows.com/?p=98>
- <https://forum.synology.com/enu/viewtopic.php?f=90&t=116726>
- <http://egloos.zum.com/voyager/v/4110197>
