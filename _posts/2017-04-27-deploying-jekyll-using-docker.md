---
layout: post
title: "docker-jekyll 설치"
tags: [jekyll, docker]
comments: true
share: true
---

지킬(jekyll)로 블로그나 웹페이지를 만들거라면 열의 아홉은 github pages를 쓰고 있을테니 이 글이 크게 도움이 되지는 않겠지만, 혹시라도 로컬에서 docker를 이용해서 설치하거나 독자적인 서버를 돌릴 경우에는 참고가 되지 않을까 싶어 적어본다.

docker를 이용하는 사례는 국내 웹에서 [두어 개](https://edeun.github.io/2017/01/17/Jekyll%EA%B3%BC-Docker%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-Github-Pages%EC%9D%98-Blog-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95.html) 찾았는데 이런 마이너 정보를 찾아볼 사람이라면 어느 정도는 지식이 있다고 가정하고 그냥 아래 ```docker-compose.yml```을 이용해서 컨테이너를 실행하면 된다.

```yml
version: '2'

services:
  jekyll:
    image: starefossen/github-pages:latest
    container_name: <your-container-name>
    restart: always
    network_mode: "bridge"
    ports:
        - 4000:4000
    volumes:
        - <your-theme-dir>:/usr/src/app
    environment:
      JEKYLL_ENV: production
```

이미지 이름에서도 알 수 있듯이 github-pages에서 사용하는 플러그인을 넘어서는 기능을 요구하는 페이지에 대해서는 에러를 뱉어낸다. 예를 들면 ```jetkyll-admin``` 그럴 경우에는 해당 항목을 추가 설치하는 ```Dockerfile```을 작성해서 직접 docker image를 빌드해서 사용하는 방법 밖에는 없다. (적어도 내가 아는 한은)

대부분 사용자 옵션이며, ```volumes``` 항목만 잘 설정하면 된다. 나의 경우에는 마음에 드는 테마 github repository를 clone해서 그 경로를 mount해 주었다.

그리고 환경변수 ```JEKYLL_ENV```가 중요한데 production으로 해주지 않으면 ```_config.yml```에 설정한 ```site.url``` 변수가 반영되지 않고 커맨드라인 옵션에서의 ```hostname```으로 되니 주의하길 바란다. ([참고](https://github.com/jekyll/jekyll/issues/5743))

마지막으로 아래 예와 같이 command를 통해서 [Dockerfile](https://github.com/Starefossen/docker-github-pages/blob/master/Dockerfile)의 마지막 ```CMD```를 override할 수 있다.

```yml
    command: >
      jekyll serve -d /_site --watch --force_polling -H 0.0.0.0 -P 4000
```

원하면 html로 publishing되는 경로인 ```-d /_site```를 외부 경로로 mount해서 nginx의 virtualhost로 줄 수도 있다. (reverse proxy를 하면 되는데 굳이 그럴 필요가...)

이제 ftp 기능이 지원되는 에디터를 이용해서 수정/저장하면 ```--watch``` 옵션을 통해서 그 내용이 실시간 반영된다.
