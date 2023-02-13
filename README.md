# mysql_errors

## persistant global connection vs connection pool
둘중 어떤 전략이 나은 판단인가? 
하나의 연결로 이루어진 지속연결인지 커넥션 풀을 이용한 트래픽 리소스 최적화인지.? 
아직 모르겠다

## 자주 볼수 있는 에러

1. Too many Connections

>
  show variables like "%max_connections%";
  
2. Error: Host 'xxx.xx.x.x' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'


## 비동기요청 관련 에러
