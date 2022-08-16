---
layout: post
title: "Gitlab Runner 연동(Docker)"
tags: [Gitlab, Gitlab Runner, CI, CD, Docker]
comments: true
---

# Gitlab - Gitlab Runner 연동(Docker 설치부터)

## Docker 설치

```bash
# 패키지 업데이트
sudo apt-get update
# https 를 통해 repository를 구성하기 위한 패키지 다운로드
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
# Docker GPG Key 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# Stable Repository 설정하기 위한 명령어 실행
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

설치가 완료된 이후 정상적으로 동작하는지를 확인한다. 

```bash
sudo docker run hello-world
```

실행 시 아래와 비슷한 메시지가 출력된다. 

```plaintext
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### Docker 디렉토리 변경하기(옵션)

도커가 용량을 많이 차지하는 경우 용량이 많은 디스크로 도커 루트 디렉토리를 변경하는 것이 좋다. 

**현재 루트 디렉토리 확인하기**

```bash
docker info | grep Root
```

**루트 디렉토리 변경하기**

예를 들어 도커 루트 디렉토리를 /data/docker 로 변경한다고 하면 아래와 같은 방법으로 변경할 수 있다. 

먼저 도커를 정지하고 디렉토리를 생성한다. 

```bash
sudo service docker stop
sudo mkdir -p /data/docker
```

다음 /etc/docker/daemon.json을 생성한다. 

```
sudo vi /etc/docker/daemon.json
```

내용은 아래와 같이 작성한다. 

```plaintext
{
    "data-root": "/data/docker"
}
```

이후 다시 도커 데몬을 시작한다. 

```bash
sudo service docker start
```

### sudo없이 docker 명령어 사용하기

도커 데몬은 기본적으로 /var/run/docker.sock에서 생성된 unix domain socket(IPC socket)을 사용하여 통신한다. 
이는 root 권한 또는 docker 사용자 그룹의 멤버의 권한을 필요로 하기 때문에 사용자가 sudo없이 docker명령을 실행하기 위해서는 docker 그룹에 사용자를 추가하면 된다. 
```bash
sudo usermod -aG docker $USER
exit
```

재로그인 하면 sudo없이 docker 명령을 실행할 수 있다. 

```bash
docker ps -a
```

## docker compose 설치 

docker compose는 복수 개의 컨테이너 실행을 정의하기 위한 툴입니다. 
자세한 내용은 다음을 참고하면 됩니다. (https://scarlett-dev.gitbook.io/all/docker/untitled)

설치 방법은 아래와 같습니다. 

```bash
sudo curl \
  -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

설치 이후 버전을 확인합니다. 

```bash
docker-compose --version
docker-compose version 1.29.2, build 1110ad01
```

## Docker 명령어 

- 도커 컨테이너 상태 확인

```bash
$ docker ps
```

- 모든 도커 컨테이너 보기

```bash
$ docker ps -a
```

- 실행중인 도커 컨테이너 bash 쉘 실행하기

```bash
$ docker exec -it [컨테이너명] bash
```


# Gitlab

## Gitlab 설치

이 문서에서는 docker-compose를 이용하여 gitlab의 관리 디렉토리를 생성하고 컨테이너를 실행시키는 방법을 서술하고 있습니다. 
일반 docker 명령을 이용하는 것은 다른 문서를 참조하시기 바랍니다. 

- 설치 디렉토리 생성

```bash
cd /data
sudo mkdir gitlab
cd gitlab
sudo mkdir data
sudo mkdir logs
sudo mkdir config
sudo chown -R $USER:$USER /data/gitlab
sudo chmod -R 755 /data/gitlab
```

- docker-compose.yml 파일 준비

```bash
vi docker-compose.yml
```

아래와 같이 내용을 추가합니다. 

```yml
version: '3.9'

services:
  gitlab:
    image: 'gitlab/gitlab-ee:14.6.2-ee.0'
    container_name: gitlab
    restart: always
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'
        gitlab_rails['gitlab_shell_ssh_port'] = 8022
        # Add any other gitlab.rb configuration here, each on its own line        
      TZ: 'Asia/Seoul'
    ports:
      - '80:80'
      - '443:443'
      - '8022:22'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
```

hostname과 external_url은 상황에 맞게 변경합니다.
(e.g. hostname: "10.11.22.33", external_url: "http://10.11.22.33")

docker-compose.yml을 생성하였다면 docker-compose를 이용하여 컨테이너를 실행합니다. 

```bash
docker-compose up -d
```

아래와 같이 구동 로그를 확인 가능합니다. 

```bash
docker-compose logs -f
```

- 초기 비밀번호 확인

```bash
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

- 접속

위에서 설정한 external_url을 웹 브라우저에서 타이핑하여 접근합니다. 

## Gitlab Runner 설치 

- 작업 디렉토리 생성

```bash
cd /data
sudo mkdir gitlab-runner
cd gitlab-runner
sudo mkdir config
sudo chown -R $USER:$USER /data/gitlab-runner
```

- docker-compose.yml 작성

```bash
vi docker-compose.yml
```

```yml
version: '3.9'
services:
  gitlab-runner:
    image: 'gitlab/gitlab-runner:latest'
    container_name: gitlab-runner
    restart: always
    volumes:
      - './config:/etc/gitlab-runner'
      - '/var/run/docker.sock:/var/run/docker.sock'
```

- gitlab runner 시작

```bash
docker-compose up -d
```

## Gitlab - Gitlab Runner 연동

1. gitlab 러너 추가 페이지 접근
	- gitlab 페이지의 Admin / Overview - Runners 페이지로 간다.

2. gitlab runner 콘솔 접근

```bash
docker-compose exec gitlab-runner bash
```

또는

```bash
docker exec -ti gitlab-runner bash
```

로 콘솔에 접근 가능하다. 

3. 러너 등록

- gitlab-runner register 명령을 실행하고 지침에 따라 아래 항목을 입력한다.

	- Enter the GitLab instance URL : external_url을 입력
	- Enter the registration token : Copy token 아이콘을 클릭하여 클립보드에 복사하여 입력
	- Enter a description for the runner : 러너에 대한 설명 입력
	- Enter tags for the runner : 공백
	- Enter an executor : docker을 입력
	- Enter the default Docker image : alpine:latest을 입력
	

4. alpine 설치

- exit를 입력하여 메인 콘솔로 복귀한 뒤
- docker pull alpine을 입력하여 alpine을 설치한다. 

5. Runner 연동 확인

Gitlab UI에서 새로고침을 하면 Runner가 등록되어있음을 확인할 수 있다. 

6. Gitlab Runner 설정

2번에서 서설한 것과 같이 다시 gitlab-runner 컨테이너에 접근 한 뒤 config.toml을 변경한다. 

```
[runners.docker] 섹션에서 privileged = true로 변경.
```
