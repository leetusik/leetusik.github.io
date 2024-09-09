+++
title = 'coupang TV[4] - 서버'
date = 2024-08-08T17:50:44+09:00
categories = ["services"]
tags = ["coupang tv", "linux", "server", "docker"]
toc = true
draft = false
+++

이번 글에서는 꿀매(쿠팡tv) 프로젝트를 배포하기위해 집 노트북 서버를 어떻게 구축했는지 소개한다. 자주 사용하지 않는 lenovo ideapad 제품을 사용했고, OS는 Fedora Linux 40를 사용했다.

<!--more-->

## 홈 서버 구축 이유

과거에 aws를 사용한 적이 있고, 또 cloudtype같은 무료 서비스 배포 플랫폼이 있지만 굳이 노트북을 활용해서 홈 서버를 구축했다. 그 이유는 비용 절감과 의존성 축소다.

### 1. 비용절감

내가 프로그래밍에 서툴러서 그런지는 모르겠지만 단순히 하루에 한 번 크롤링 프로그램이 약 45분 정도 돌아가는 꿀매(쿠팡tv)같은 작은 서비스도 메모리 2 GB 정도는 상시 사용한다.
{{< image_caption src="memory.png" alt="홈 서버 메모리 사용 현황" caption="홈 서버 메모리 사용 현황" >}}

aws의 경우에는 EC2를 사용했을 때, 2 GB는 `t3.medium`이상 플랜을 사용해야하고, 이건 달에 $30도 넘는다.
cloudtype의 경우에도 프로 플랜이 1 GB 램 기준 2 만원부터 시작이고, 추가 RAM을 사용하려면 추가 비용을 내야한다.

당장 사용자가 있을 것으로 기대되지 않는 내 서비스의 특성과 앞으로도 다수의 서비스를 추가로 만들 것을 생각할 때, 당장의 서버 안정성보다 당장의 주머니 사정을 고려했다.

지금 당장 전기 요금 수준은 잘 모르겠지만, 전기 요금이 150원/kWh 이라고 가정했을 때, 한 달 요금이 약 3 천원이라는 챗 지피티의 계산이 나온다. 이는 클라우드 서비스를 이용할 때에 비해 1/10 수준이며 컴퓨터 성능도 훨씬 좋다!

### 2. 의존성 축소

요즘에는 aws의 elastic beanstalk처럼 거의 파일만 옮기면 바로 배포해주는 서비스가 많은 것 같다. 하지만 서버가 어떻게 돌아가는지 이해해야(적어도 조금은 알아야) 추후에 스케일을 확장하거나 혹은 축소하고 싶을 때 유동적인 대처가 가능하겠다는 판단이 들었다.

---

## fedora linux 40 환경 설정

[유튜브 TechHut](https://youtu.be/HxvFuGnjoJo?si=m7XG6NQbWbXLbftY)의 동영상으로 서버 구축의 대부분을 배웠다.

### 1. live USB 만들기

1.  [Fedora Server 40 Download](https://fedoraproject.org/server/download)

    - 본인 컴퓨터에 맞는 iso 파일을 다운로드한다.

2.  [balenaEtcher Download](https://etcher.balena.io/)

    - balena 프로그램을 다운로드 한 후, iso 파일을 사용해 Live USB를 만든다.

### 2. 리눅스 설치 및 설정

USB를 노트북에 꼽고 부팅할 때 바이오스 설정으로 들어가서 USB로 부팅을 시도한다. 이후 설치 유도에 따라 진행한다.

#### 2.1 저장공간 활용 최대로

fedora linux를 처음 설치했을 때, 이미 윈도우가 깔려있는 디스크에 파티션을 구분해서 설치를 해서인지, 60 GB의 파티션에 설치를 했음에도 불구하고 15 GB의 공간만 사용할 수 있는 것을 확인했다.

아래의 명령어로 60 GB 전체 공간을 활용할 수 있었다.

```bash
sudo lvextend -l +100%FREE /dev/mapper/fedora-root
```

#### 2.2 FileZilla

[FileZilla](https://filezilla-project.org/)를 다운받아서 같은 네트워크에만 연결되어 있다면 다른 기기간 파일을 자유롭게 옮길 수 있다.

#### 2.3 노트북 닫은 채로 실행

`/etc/systemd/logind.conf` 파일을 아래와 같이 설정해서 노트북을 닫은 채로 서버를 돌릴 수 있다.

```conf
# /etc/systemd/logind.conf 파일 설정

[Login]
HandleSuspendKey=ignore
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
```

#### 2.3 Docker 설치

리눅스 터미널에서 아래와 같은 명령어를 활용해 docker를 설치할 수 있다.

```bash
curl -fsSL https://get.docker.com -o get-docdekr.sh
sudo sh get-docker.sh
```

---

## 꿀매(쿠팡tv) 배포

docker까지 설치했다면 기본적인 배포 준비는 끝났다. 서버 설치부터 서비스 배포까지 모두 처음해봤기 때문에 상당한 시간이 들었다.

### 1. 프로젝트 클론

프로젝트를 복사할 디렉토리로 이동한 뒤에 다음의 명령어를 입력한다. 쿠팡tv 서비스를 만들 때 suganglive 깃헙 계정을 사용하여 작업했기 때문에 그 repository를 가져온다.

```bash
git clone --branch feature/coupangman2 https://github.com/suganglive/coupangman.git
```

### 2. docker 설정

그냥 `flask run`을 입력하면 되는 로컬 환경과 다르게, docker에서 앱을 실행하려면 2가지 + 1가지 파일을 따로 생성해야한다.

#### 2.1 Dockerfile 설정

첫 번째는 Dockerfile이다. 확장자 명 없이 그냥 Dockerfile.

```Dockerfile
# Dockerfile

# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Install Chromium and other dependencies
RUN apt-get update && \
    apt-get install -y chromium chromium-driver tzdata && \
    apt-get clean

# Set the timezone to Asia/Seoul
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone


# Set the working directory in the container
WORKDIR /app


# Copy the current directory contents into the container at /app
COPY . /app

# Install setuptools and pip
RUN pip install --upgrade pip setuptools

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Make port 5000 available to the world outside this container
EXPOSE 5000

# Define environment variable
ENV NAME World

# Run run.py when the container launches
CMD ["python", "run.py"]
```

Chatgpt의 힘으로 작성한 코드이기 때문에 영어로 주석처리가 되어있다.

#### 2.2 docker-compose.yml 설정

```yml
# docker-compose.yml

version: "3.8"

services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - FLASK_ENV=development

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  celery_worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - web
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  celery_beat:
    build: .
    command: celery -A celery_worker.celery beat --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - web
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    depends_on:
      - web

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: /bin/sh -c "trap exit TERM; while :; do certbot renew; sleep 12h & wait $$!; done;"
```

https도 설정하고싶어서 certbot을 추가했지만, 실패했다. 지금 꿀매(쿠팡tv)는 사용자가 보안에 유의할 정도의 정보 입력을 필요로하지 않으니, https는 다음으로 미루기로 했다.

아래의 두 커맨드는 원래 터미널에서 직접 입력하던 것들인데, 이런 것을 처리하는게 docker-compose.yml의 역할이구나 싶었다.

```bash
celery -A celery_worker.celery worker --loglevel=info
celery -A celery_worker.celery beat --loglevel=info
```

#### 2.3 nginx.conf 설정

솔직히 이해가 되는 부분은 많지 않다. 그냥 작동하니 내비 뒀다. https는 실패했다. 많이 하다보면 자연스레 알게 되겠지 싶다.

```conf
user  nginx;
worker_processes  auto;
error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen 80;
        server_name ggulmae.com www.ggulmae.com;

        location / {
            proxy_pass http://web:5000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        error_page 404 /404.html;
        location = /404.html {
            internal;
        }
    }
}
```

#### 2.4 기타 커맨드

아래의 커맨드로 docker에서 돌아가는 앱을 재시작한다. Dockerfile 등을 수정하지 않았다면 build 라인은 빼도 된다.

```bash
sudo docker-compose down
sudo docker-compose build
sudo docker-compose  up -d
```

다음의 커맨드로 리눅스 터미널에서 실행중인 docker web 서비스에서 `python` 명령어를 통해 파이썬에 접근할 수 있다. 파이썬에 접근해서 tasks.py에 정리된 task를 실행할 수 있다.

celery beat를 사용해 매일 5시 17분(그냥 시험해보던 시간이 5시 언저리였기 때문이다.) 스크래핑을 하지만, 시험하는 과정에서 이렇게 수동으로 파이썬에 많이 접근했다.

```bash
sudo docker-compose exec web /bin/sh

```

### 3. 로그 확인

docker에서 돌아가는 파일 확인

```bash
sugang@localhost:~/projects/coupangman$ sudo docker-compose ps
NAME                         COMMAND                  SERVICE             STATUS              PORTS
coupangman-celery_beat-1     "celery -A celery_wo…"   celery_beat         running             5000/tcp
coupangman-celery_worker-1   "celery -A celery_wo…"   celery_worker       running             5000/tcp
coupangman-certbot-1         "/bin/sh -c 'trap ex…"   certbot             running             80/tcp, 443/tcp
coupangman-nginx-1           "/docker-entrypoint.…"   nginx               running             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
coupangman-redis-1           "docker-entrypoint.s…"   redis               running             0.0.0.0:6379->6379/tcp, :::6379->6379/tcp
coupangman-web-1             "python run.py"          web                 running             0.0.0.0:5000->5000/tcp, :::5000->5000/tcp
```

celery_worker의 로그 확인(마지막 50줄)

```bash
sugang@localhost:~/projects/coupangman$ sudo docker-compose logs --tail=50 celery_worker
```

### 4. 기타 에러 해결

http://ggulmae.com 에 접근했을 때, 주소 뒤로 이상한 주소가 붙으면서 http://ggulmae.com/login/login.cgi 주소로 접속되는 오류가 있었다.
구글링을 통해 네트워크 포트 충돌 이슈일 가능성이 있다는 것을 확인하고, iptime 포트포워드 설정에서 새로운 규칙을 추가하여 문제를 해결했다.

{{< image_caption src="network setting.png" alt="포트포워드 규칙" caption="포트포워드 규칙" >}}

#### 수정 필요 사항

1. https 추가

#### 같은 주제 게시글 모음

{{< related_posts tag="coupang tv" >}}
