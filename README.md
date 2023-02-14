# 1. persistent global connection vs connection pool
둘중 어떤 전략이 나은 판단인가? 
하나의 연결로 이루어진 지속연결인지 커넥션 풀을 이용한 트래픽 리소스 최적화인지.? 
## 1.1 DB connection Pool 을 지양하는 이유
보통 Connection Pool을 사용하는 이유에는 Connection에 드는 비용이 크기 때문이라고 한다  
애플리케이션에서 매쿼리마다 connection을 생성하고 종료한다면 어떤 것에 비용적으로 비효율적인지 알아보자

## 1.1.1 Network
DBMS 와의 통신은 TCP/IP로 이루어지는데 3-way-handshake 과정을 거치고  
가상경로의 연결은 지속 유지되며 연결을 끊는 과정에서 다시 4-way-handshake 를 거친다.  
이러한 연결 과정을 매번 반복하지 않을수 있는데 반복한다는건 비효율적인 일이다.

## 1.1.2. Oracle DBMS
오라클과 클라이언트의 연결과정
1. 오라클 리스너 시작 (LISTEN)
2. 애플리케이션에서의 커넥션 시도 
- 리스너와 클라이언트 사이에 소켓 생성
3. 서버 프로세스의 생성
- 서버 프로세스를 생성하여 SQL 처리를 즉시 인계
- 리스너가 인계한 후부터 서버 프로세스와 오라클 클라이언트는 직접 송수신 하므로 리스너는 자유로워 진다
- 애플리케이션에서 접속을 종료하는 처리(close, disconnect)를 수행하면 서버 프로세스도 함께 종료된다  

오라클의 경우 연결마다 매번 서버 Process를 실제로 OS단에 생성해야 한다   
이는 공유 메모리 확보등 무거운 작업이다. 이 과정을 반복하는것은 엄청난 비효율이다  
오라클은 하나의 쿼리마다 하나의 Process가 생기게 되어 부담이 크다  

## 1.1.3. MySQL DBMS
MySQL은 단일 프로세스, 멀티 스레드 개념이다  
오라클과 달리 클라이언트를 담당할 Process가 아닌 Thread가 존재한다.  
MySQL 서버로 요청이 들어올때 마다 배정되는 Foregorund Thread 도 있고 계속 돌고 있는 Background Thread 도 있다.  
갑자기 요청이 천개가 들어왔다고 가정 하에 Thread 가 그 수만큼 늘어나게 되면 Context Swtiching 관련 리소스가 커져서 시스템 성능이 저하된다.  
따라서 Multi Thread 프로그램들은  Thread Pool을 이용해서 Thread의 수를 제한한다.  

1. Foreground Thread  
Background Thread 와는 다르게 기본적인 차이점은 Foregroun Thread 는 메인 Thread가 종료 되더라도 Foreground Thread 가 살아 있는 한  
Process 가 종료되지 않고 계속 실행되며 Background Thread 는 Main Thread 가 종료되면 바로  Process 를 종료한다  
- 이 Thread 는 사용자가 요청한 쿼리문장을 처리하는 역할을 한다
- 이러한 Thread 도 매 연결마다 생성하면 부담이 크므로 Tread Pool 개념을 사용한다
- 실제 동시접속이 늘었을때 Thread 생성으로 인해 지연이 발생할수 있다

2. Connection Pool  
![image](https://user-images.githubusercontent.com/41939976/218619108-c5ffd9ef-6e15-4a01-ba94-10c28c9ee693.png)
- 소프트웨어 엔지니어링에서 Connection Pool은 데이터베이스에 대한 향후 요청이 필요할때 연결을 재사용할 수 있도록 유지관리하는 데이터베이스 연결의 캐시다
- Connection Pool은 데이터베이스에서 명령을 실행하는 성능을 향상시키는데 사용된다
- 각 사용자에 대한 데이터베이스 연결, 동적 데이터베이스 기반 웹사이트에 대한 요청 비용은 많이 들고 자원을 낭비한다
- Coonection pooling에서는 연결이 생성된후 pool에 배치하고 다시 사용되므로 새 연결을 설정할 필요가 없다
- 모든 Connection을 사용하는 경우 새 연결이 만들어지고 pool에 추가된다
- 데이터베이스에 대한 연결을 설정하기 위해 사용자가 기다려야 하는 시간을 줄여주는데 thread_cache_size와 개념이 비슷하다  

2.1. 주의사항
- pool에 최대로 저장되는 connection 수는 정해져 있고 요청이 많은 경우 connection이 모두 사용중이라면 반납될때까지 대기를 한다
- connection 수가 적으면 안되지만 많이 늘리게 되면 메모리를 많이 사용하게 되어 성능을 저하 시킬수 있다
- 이용자 수에 따라 connection 수를 적절하게 지정해야 한다
- 사용자 요청수가 적고 동시 접속이 거의없는 경우에는 매 쿼리마다 connection을 빠르게 맺고 끊것이 오히려 성능상 이점이다

3. Thread Pool 
- 컴퓨터 프로그래밍에서 Thread Pool은 컴퓨터 프로그램에서 실행의 동시성을 달성하기 위한 소프트웨어 설계 패턴이다 (복제된 작업자, 스크루 모델이라고도 불림)
- 사용 가능한 Thread 수는 실행 완료 후 병렬 작업 대기열과 같이 프로그램에서 사용할수 있는 컴퓨팅 리소스에 맞춰 조정된다
- MySQL 은 Foreground Thread 를 미리 생성하여 대기시켜 놓는데, 대기하고 있는 Thread 들을 모아둔 곳이 Thread Pool 이다  
- 최소한 서버에 접속된 클라이언트 수만큼 존재해야 한다
- 사용자가 DB Connection 을 종료하면 해당 Thread는 Thread Pool로 돌아간다
- 작업은 내부적으로 Queue에 저장되며(Thread 작업 할당) 새로운 작업이 들어올때마다 저장된 작업을 수행한다. 작업이 할당되지 않은 Thread는 대기상태로 유지된다
- 예를 들어 Thread Pool의 Thread 최대 갯수가 10 이면 갑자기 요청이 100이 올경우 처음 10개의 요청에 대해서는 Thread를 배정하고 나머지는 Queue에 넣어 대기시킨다 
- Connection(Thread)를 미리 생성해놓고 재사용하면서 DB 트랜젝션을 관리하여 부하를 줄이고 Connection pool의 개념에서 부하를 관리할 수있게 좀 더 나아간 버전이다
- 참고할 옵션으로는 thread_pool_size, thread_cache_size 

## 1.1.3 Application (Nodejs)
Nodesjs 에서 애플리케이션 시작시 Connection 을 5개 생성하여 Connection Pool을 구성한다고 가정할 경우  
1. 애플리케이션이 시작되고 5개의 Connection 생성
2. 각 Connection은 TCP/IP 3-way-handshake 과정을 거치고 Nodejs의 Socket은 각각 서로 다른 고유한 포트번호에 할당된다
3. 연결은 유지되며 즉 5개의 서로 다른 Socket이 열려있다
4. 이후 이미 형성된 Connection(Socekt) 을 Pool의 Queue 에서 재사용된다

# 2. 흔히 접할만한 Mysql 에러들과 
## 2.1. Error: Too many Connections

MySQL 에 허용 동시 접속자수가 다차서 더이상 접속할 수없다는 에러이다

```sql
show variables like "%max_connections%";
```
![image](https://user-images.githubusercontent.com/41939976/218401943-6e4ff942-64b2-4699-9950-94a929710ecc.png)
- max_connections : 최대 접속수 (기본값 151)

```sql
show status like "%connect%";
```
![image](https://user-images.githubusercontent.com/41939976/218405817-4ae100a4-dfa0-4b3d-8d67-f03cae92fe0a.png)
- Aborted_connects: MySql 서버에 접속이 실패된 수
- Connections: 연결된 Thread 수
- Max_used_connections: 최대로 동시에 접속한 수
- Threads_connected: Thread Cache의 Thread 수

```sql
show status like '%thread%';
```
![image](https://user-images.githubusercontent.com/41939976/218406185-24f2544f-8ff2-4f15-b9ad-9c12b1b30b19.png)
- Threads_connected: 현재 연결된 Thread 수
- Threads_created: 접속을 위해 생성된 Thread 수
- Threads_running: Sleeping 되어 있지 않은 Thread 수

위에 상태값들을 토대로 Rate를 도출하여 Too many Connection 에 대한 문제 해결법을 찾는다.

1. Cache Miss Rate
```
Cache Miss Rate(%) = Threadscreated / Connections x 100
```
- Cache Miss Rate(%) 이 높다면 thread_cache_size 를 기본값인 8 보다 높게 설정하는 것을 권장
2. Connection Miss Rate
```
Connection Miss Rate(%)= Abortedconnects / Connections x 100
```
- Connection Miss Rate(%) 이 1% 이상 된다면 wait_timeout 을 좀더 길게 가져가는 것을 권장
3. Connection Usage
```
Connection Usage(%) = Threads_connected / max_connections x 100
```
- Connection Usage 가 100% 이거나 높다면 max_connections 수를 증가시키는 것을 권장
- connection 수가 부족할 경우 Too many connections 에러가 발생한다

### 2.1.1 해결방안
MySQL 에 접속하여 설정값을 직접 변경하거나 환경설정 파일(my.cnf)에서 변경해주는 방법으로 크게 두가지가 있다.   

1. max_connections 변경
- mysql.conf 파일의 내용에서 해당 옵션을 적당한 값으로 늘려서 변경한다
2. wait_timeout 변경
- wait_timeout은 mysqld와 client가 연결을 맺고 다음 쿼리까지 기다리는 최대시간이다
- wait_timeout 안에 요청이 들어올 경우 0으로 초기화 된다
- too many connection 을 우회하기 위해선 기본값이 28800초 보다 줄여야 한다. 트래픽이 많을 경우 30초 정도로 적게 하는것이 좋다  
- 다만 connection 을 전역 변수로 이용하는 경우 wait_timeout 시간이 지나면 재연결을 할수가 없으므로 connection pool 이용을 권장한다  
3. connection pool 이용시 connection limit 변경
- max_connections 와 맞게 변경 해준다
4. interactive_timeout 변경
- interactive_timeout은 터미널 모드 에서의 다음 쿼리까지 기다리는 최대 시간이다. (wait_timeout 과 비슷)
- 이것도 또한 wait_timout 과 동일하게 기본값이 28800초 이므로 3600초로 하는것을 권장

```sql
show processlist;
select * from information_schema.processlist where command != 'sleep';
```
해당 사항을 체크후 변경한 후에 processlist 를 확인한다

## 2.2. Error: Host 'xxx.xx.x.x' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'

시스템은 멀쩡한데 느닷없이 mysql 서버로부터 접속 거부를 당한다. 이 에러의 max_connect_errors와 연관성이 높다.  
MySQL 서버가 동작중인지 원격에서 검사하거나 해당 원격 서버의 포트가 살아있는지 검사할때 단순히 커넥션을 한후 close 하게 되면  
MySQL은 비정상적인 접속으로 판단하여 해당 IP를 블럭킹한다고 한다.  
필자의 경우 서비스 요청 마다 단일 커넥션을 한후 쿼리를 끝낸후에 커넥션을 끊었더니 언제부턴가 갑자기 서버에서 mysql 원격지로 부터 ip를 차단당하는 일이 발생했다.  
따라서 나의 경우 아예 단일 커넥션의 현상을 줄이기 위해 코드적으로 전역적으로 단일 연결을 해서 문제를 해결했다.  
하지만 단일 연결에서의 문제가 있다면 8시간이상(wait_timeout 기본값) 요청이 없을 경우 스스로 연결이 해제되어 reconnect 가 불가능했다.  
해당 문제가 있던 서버는 API 서버이고 트래픽이 많은 편이라 장애가 될만한 부분은 아니었다.   

MySQL은 비정상적인 접속에 대한 요청수를 카운트 하는데 max_connect_errors 변수에서 지정한 값을 넘으면 블럭킹한다.  
기본값은 10이며 정기적인 포트 점검이 필요한 경우 이수를 높이라고 권장한다.  
MySQL 메뉴얼에 따르면 아래와 같이 권장한다. 

### 2.2.1 MySQL Manual

- MySQL의 안정성은 DNS의 안정성에 달려있다. 즉, DNS의 설정 상태가 좋지 않을 경우에는 MySQL 서버도 그만큼 안정하지 못하게 된다. 
- 만일 네트워크가 안정하지 못할 경우, max_connect_errors가 빠른 시간안에 10(default value)에 도달하게 되고 동일 클라이언트의 향후 커넥션을 거부하게 된다.  

따라서 이러한 문제를 해결 하기 위한 방법을 아래와 같이 제안한다.  
1. 절대로 외부에서 MySQL 서버에 커넥션을 하도록 만들지 말 것
2. MySQL 에서 libwrap를 지원하도록 활성화 시키지 말 것. 이것을 활성화 시키는 것은 단지 문제를 더 어렵게 하는 것이다.
3. 사용자의 my.cnf 파일에서 skip_name_resolve를 활성화 시킬 것. 이렇게 하면 모든 호스트 이름에서 점(period)를 비활성화(disable) 시킨다.  
- 모든 ALL GRANT는 반드시 IP주소를 기반으로 되어 있어야 한다.
4. max_connect_errors를 매우 높게 설정한다 (ex value: 99999999)
- 이렇게 하면 네트워크 또는 클라이언트의 문제로 인한 우발적인 커넥션 단절 문제를 피할 수가 있다.  

## 참고문서
* [스레드가 너무 많으면 성능이 저하되는 이유와 해결 방법](https://www.codeguru.com/cplusplus/why-too-many-threads-hurts-performance-and-what-to-do-about-it/)
* [Thread Pools](https://parkcheolu.tistory.com/30)
* [thread_pool_size, thread_pool_oversubscribe](https://runebook.dev/ko/docs/mariadb/thread-pool-system-and-status-variables/index)
* [하드웨어 스레드와 소프트웨어 스레드](https://juneyr.dev/thread)
* [멀티코어 프로그래밍에서 흔히 발생하는 문제 1부](https://andromedarabbit.net/멀티코어-프로그래밍에서-흔히-발생하는-문제-1부/)
* [내가 만든 서비스는 얼마나 많은 사용자가 이용할 수 있을까?](https://hyuntaeknote.tistory.com/12)
* [MySQL 8.0 Reference Manual > 5.6.3.4 Thread Pool Tuning](https://dev.mysql.com/doc/refman/8.0/en/thread-pool-tuning.html)
* [mysql persistent connection vs connection pooling](https://stackoverflow.com/questions/9736188/mysql-persistent-connection-vs-connection-pooling)
