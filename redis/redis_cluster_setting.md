# Redis cluster 구성

Redis는 오픈소스(BSD 라이센스), 메모리 내 데이터 구조 저장소입니다. Memcached 대안으로 사용하여 간단히 키-값 쌍을 저장 할 수 있습니다.<br>
또한 NoSQL 데이터베이스 또는 Pub-Subj 패턴의 메시지 브로커로도 사용할 수 있습니다.<br>
<br>
Amazon Elastic Cache 를 사용해서 cluster를 구성하는 경우 설정이 따로 필요없습니다.<br>

- Redis를 위한 충분한 여유 메모리가 있어야 합니다
- Redis용 새 Ubuntu 서버를 배포하는 경우 Redis 서버와 애플리케이션 서버 간에 사설 네트워크를 활성화 및 구성해야 합니다(클라우드 경우 VPC)

## Ubuntu 서버 업데이트
1. 활성화된 저장소에서 패키지 목록을 업데이트
```sh
$ sudo apt update && sudo apt upgrade -y
```
2. 서버 다시 시작
```sh
$ sudo reboot
```

## UFW 활성화 (선택사항)

운영하는 서버에 인바운드 아웃바운드에 대한 보안 설정이 안되어 있다면 하는것을 추천<br>
1. 기본 규칙 세트로 UFW 활성화
```sh
$ sudo ufw enable
```
2. UFW 상태 체크
```sh
$ sudo ufw status
```
> ufw: command not found (UFW 미설치 상태)
> Status: inactive (UFW 설치가 되어있으나 미구성 상태)
> Status: active (UFW 실행 상태)

- UFW 비활성화
```sh
$ sudo ufw disable
```
- UFW 기본값으로 재설정
```sh
$ sudo ufw reset
```

```
필요한 서버 3대: Ubuntu 20.04 LTS
각 서버 master node(:7000) 1개
각 서버 slave node(:7001) 1개
```

cluster 구성시 cluster bus가 통신하는 포트는 각 node 포트 + 10000번이기 때문에<br>
총 7000, 7001, 17000, 17001번 포트의 방화벽을 해제해야 한다.<br>
- cluster bus: 장애감지, 구성 업데이트, fail over 승인 등에 사용

## 1. Redis 설치

- 설치 후 redis-server 명령어를 입력했을때 서버가 잘 뜬다면 설치 완료

>### 1.1. wget case
```bash
wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
make install
```

>### 1.2. apt package case
```bash
sudo apt update
sudo apt install redis-tools
sudo apt install redis-server
server redis-server start 또는 stop
redis-cli로 연동
```

>### 1.3. 환경변수 설정
```bash
# 메모리 사용량이 허용량을 초과할 경우, overcommit을 처리하는 방식 결정하는 값을 "항상"으로 변경
sudo sysctl vm.overcommit_memory=1
sudo echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
sudo echo vm.overcommit_memory=1 | sudo tee -a /etc/sysctl.conf 안되면 이거쓰셈
sudo sysctl -a | grep vm.overcommit

# 서버 소켓에 Accept를 대기하는 소켓 개수 파라미터를 변경
sudo sysctl -w net.core.somaxconn=1024
sudo echo "net.core.somaxconn=1024" >> /etc/sysctl.conf
sudo echo net.core.somaxconn=1024 | sudo tee -a /etc/sysctl.conf
sudo sysctl -a | grep somaxconn
```

>### 1.4. Direcotry 구조
/opt/redis 디렉토리를 생성하여 redis 설치 디렉토리의 redis.conf 파일을 /opt/redis에 복사.
redis 설치 디렉토리를 그대로 입력했다면 redis-stable일 것이다.
/opt/redis 아래에 7000이라는 폴더를 생성한다 (/opt/redis/7000)

>### 1.5. master node config 파일 생성

- /opt/redis/7000 에 7000.conf 파일을 생성한다.

```bash
# 7000.conf
bind 0.0.0.0
daemonize yes
protected-mode no

port 7000
pidfile /opt/redis/7000/redis_7000.pid
logfile /opt/redis/7000/redis-7000.log
dir /opt/redis/7000
dbfilename dump_7000.rdb

requirepass [redis 비밀번호]
masterauth [redis 비밀번호]

cluster-config-file node-7000.conf
cluster-enabled yes
cluster-node-timeout 5000
cluster-announce-ip [서버 public ip] # NAT/포트포워딩 사용시

rename-command keys ""

appendonly no
```