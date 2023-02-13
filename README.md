# mysql_errors

## persistant global connection vs connection pool
둘중 어떤 전략이 나은 판단인가? 
하나의 연결로 이루어진 지속연결인지 커넥션 풀을 이용한 트래픽 리소스 최적화인지.? 
아직 모르겠다

# 1. 자주 볼수 있는 에러
## 1.1. Too many Connections

```sh
show variables like "%max_connections%";
```
![image](https://user-images.githubusercontent.com/41939976/218401943-6e4ff942-64b2-4699-9950-94a929710ecc.png)
- max_connections : 최대 접속수 (기본값 151)

```sh
show status like "%connect%";
```
![image](https://user-images.githubusercontent.com/41939976/218405817-4ae100a4-dfa0-4b3d-8d67-f03cae92fe0a.png)
- Aborted_connects: MySql 서버에 접속이 실패된 수
- Connections: 연결된 Thread 수
- Max_used_connections: 최대로 동시에 접속한 수
- Threads_connected: Thread Cache의 Thread 수

```sh
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

## 1.2. Error: Host 'xxx.xx.x.x' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'
>
  asdf


## 비동기요청 관련 에러
