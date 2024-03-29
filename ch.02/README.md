## 목차

2-1. [리버스 프록시(Reverse Proxy) 도입](#2-1-리버스-프록시reverse-proxy-도입)

2-2. [캐시서버 도입](#2-2-캐시서버-도입)

2-3. [MySQL 리플리케이션](#2-3-mysql-리플리케이션)

2-4. [MySQL 슬레이브 활용](#2-4-mysql-슬레이브-활용)

2-5. [고속, 경량 스토리지 서버 선택](#2-5-고속-경량의-스토리지-서버-선택)

번외. [아파치 모듈을 이용한 리버스 프록시](https://github.com/revenge1005/System-Redundancy/tree/master/ch.02/%5B%EB%B2%88%EC%99%B8%5D%20%EC%95%84%ED%8C%8C%EC%B9%98%20%EB%AA%A8%EB%93%88%EC%9D%84%20%EC%9D%B4%EC%9A%A9%ED%95%9C%20%EB%A6%AC%EB%B2%84%EC%8A%A4%20%ED%94%84%EB%A1%9D%EC%8B%9C)

<br>

---

<br>

# 2-1. 리버스 프록시(Reverse Proxy) 도입
> IPVS(LVS)와 같은 Load Balancer는 L4 레벨에서 패킷을 전송하기만 할 뿐, 웹 서버가 클라이언트으로부터의 요청에 직접 응답하는 구성이라는 점은 변함이 없다. 여기서 Load Balancer 대신 Reverse Proxy 서버를 넣음으로써 보다 유연하게 부하를 분산할 수 있게 된다. <br><br>
Reverse Proxy는 클라이언트로부터의 요청을 받아서 웹 서버로 요청을 전송하면, 웹 서버는 요청을 받아서 처리를 하지만 응답은 클라이언트로 보내지 않고 Reverse Proxy로 반환한다. 요청을 받은 Reverse Proxy는 그 응답을 클라이언트로 반환한다.

<br>

## ⓐ 프록시(Proxy) 서버
> 클라이언트가 자신을 거쳐 다른 네트워크에 접속할 수 있도록 중간에서 대신 전달해주는 서버

<br>

#### 【 Forward Proxy 】
> 클라이언트에서 요청을 할 때 직접 요청하는 것이 아닌 Proxy 서버를 거치는 방식 

<br>

+ **보안**

    - [x] 클라이언트는 Proxy를 통해서만 외부에 요청을 하기 때문에 특정 사이트에 직접적으로 접근하는 것을 방지할 수 있다.<br>

+ **캐싱**

    - [x] 클라이언트의 요청을 Proxy가 캐싱하여 동일한 요청이 들어온 경우 캐싱된 데이터를 전달함으로써 서버의 부하를 줄일 수 있다.<br>

+ **암호화**
    
    - [x] 클라이언트의 요청은 Proxy를 통해 암호화되어 다른 서버를 통과할 때 필요한 최소한의 정보만 갖게 되는데, 이는 클라이언트의 IP를 숨길 수 있다.

<br>

#### 【 Reverse Proxy 】
> 서버에서 클라이언트로 직접 데이터를 전달하지 않고 프록시 서버를 거치는 방식 

<br>

+ **로드밸런싱**

    - [x] 여러 대의 서버에 트래픽을 분산시켜 서비스의 퍼포먼스와 신뢰성을 향상하기 위한 서비스 <br>

+ **보안**   

    - [x] 서버의 IP를 노축시키지 않을 수 있기 때문에 서버에 대한 1차적인 공격을 막을 수 있다. <br>

+ **캐싱**

    - [x] Proxy 서버에 캐싱되어 있는 데이터를 사용하여 클라이언트에 대한 요청을 처리 <br>

+ **SSL Offloading(or Termination)**

    - [x] **Proxy 서버가 SSL을 전담해서 처리하고 서버에게 흘려보내는 기능** <br><br>
    - [x] Proxy 서버가 SSL 암호화를 해제하고 서버는 암호화가 해제된 데이터를 전달하는 것<br><br>
    - [x] SSL Passthrough - 특별한 액션 없이 그대로 흘려보내는 기능 <br>

+ **압축**
    - [x] 서버의 응답을 클라이언트로 반환하기 전에 압축하면 필요한 대여폭이 줄어들어 전송 속도 향상 

<br>

## ⓑ 리버스 프록시 구체적인 이점/기능

<br>

#### 【 HTTP 요청 내용에 따라 시스템의 동작 제어 (L7 스위치가 하는 역할과 동일함) 】
> Reverse Proxy가 있으면 HTTP 요청 내에서 URL 보고 최종적인 처리를 각기 다른 서버에 분배하는 제어가 가능해진다.

<br>

#### 【 시스템 전체의 메모리 사용효율 향상 】
> 정적 컨텐츠만 반환하는 서버에 비해 동적 컨텐츠를 반환하는 서버에서는 수 배의 메모리를 소비하는 것도 드물지 않다. **서버는 클라이언트의 하나의 요청에 대해 하나의 프로세스/쓰레드를 할당해서 처리하는 방식을 취하고 있으며 정적 컨텐츠를 반환하는, 즉 파일에 쓰인 내용을 그대로 반환하기만 하면 될 경우도 동일한 방식으로 반환하게 된다.**

<br>

+ **동적으로 생성된 하나의 HTML 페이지 내에 이미지가 30개 정도 사용되고 있다고 가정할 때** <br><br>
    - [x] 이 페이지에 대한 요청은 첫 요청 한 번만 동적 컨텐츠를 요구하게 되고 첫 요청으로 HTML 동적으로 생성되며 이 HTML은 클라이언트인 브라우저에 의해 다운로드 된다. <br><br>
    - [x] 브라우저는 HTML을 해석해서 필요한 이미지 파일이나 스크립트 파일을 서버에 요청한다. <br><br>
    - [x] 결과적으로 동적 요청 (1회) + 정적 요청 (30회)가 된다. <br><br>
+ **모두 AP 서버에서 응답할 경우** <br><br>
    - [x] 모두 AP 서버에서 응답할 경우, 단 1회 동적 요청을 처리할 뿐임에도 나머지 30회의 정적인 요청에 대해 응답할 때에도 대량의 메모리를 소비하게 된다. <br><br>
    - [x] 이는 이미지든 동적 컨텐츠든 똑같이 하나의 요청에 대해 하나의 프로세스/쓰레드로 응답할 필요가 있기 때문이다. <br><br>
+ **서버를 분할할 경우** <br><br>
    - [x] 정적 컨텐츠는 메모리 소비량이 적은 웹 서버가 응답하고, 동적 컨텐츠만 애플레킹션으로 응답하는 형태의 구성이 가능해진다. <br><br>
    - [x] Reverse Proxy를 통해 정적 컨텐츠, 동적 컨텐츠에 대한 요청을 각각의 서버로 할당할 수 있다 (ex. URL의 내용을 보고 요청을 할당할 곳을 변경한다.) <br><br>
    - [x] 이때, Reverse Proxy 자신도 웹 서버라는 특징을 살려서 (정적 컨텐츠를 반환하기 위해 웹 서버를 별도로 준비하는 것이 아니라) 정적 컨탠츠는 Reverse Proxy 자신이 반환하는 형태의 구성이 일반적이다.

<br>

#### 【 웹 서버가 응답하는 데이터의 버퍼링의 역할 】
+ **HTTP의 Keep-Alive** <br><br>
    - [x] 특정 클라이언트가 한 번에 다수의 컨텐츠를 동일한 웹 서버로부터 얻고자 할 경우, 다수의 HTTP 요청마다 서버와 접속하고 끊고를 반복하는 것은 비효율적이다. <br><br>
    - [x] 최초 요청 시 연결된 서버와의 접속을 해당 요청이 종료한 후에도 접속을 끊지 않고 유지한 채로 이어지는 요청에 해당 접속을 계속 사용함으로써, 하나의 접속으로 다수의 요청을 처리할 수 있도록 실현한 것이 Keep-Alive이다. <br><br>
    - [x] Kepp-Alive는 한번 연결된 접속을 당분간 유지하는 특성상, 웹 서버에 부하를 야기한다 <br><br>
    - [x] 구체적으로는 특정 클라이언트로부터 요청을 받은 프로세스/쓰레드는 그 시점으로부터 일정 시간 동안 해당 클라이언트로의 응답을 위해서 점유되는 것을 들 수 있다. <br><br>
+ **메모리 소비와 Keep-Alive ON/OFF** <br><br>
    - [x] 하나의 프로세스당 메모리 소비량이 많은 AP 서버에서는 하나의 호스트 내에 실행될 수 있는 최대 프로세스 수는 50 ~ 100개 정도다.  <br><br>
    - [x] 이때 **리버스 프록시 없이 Keep-Alive를 설정한 경우 50~100개의 프로세스 중 다수가 Keep-Alive의 접속 유지를 위해 소비되고 Keep-Alive를 OFF로 설정하면 클라이언트에서 볼 때의 체감속도가 저하**된다. <br><br>
    - [x] 리버스 프록시 도입한 경우를 생각해 보면, 일반적으로 리버스 프록시 역할을 하는 웹 서버는 프로세스당 메모리 소비량이 그다지 많지 않으므로 하나의 호스트 내에 1000~10000 프로세스를 실행할 수도 있다. <br><br>
    - [x] 이 경우에는 일부 프로세스가 Keep-Alive 연결을 유지하기 위해 소비된다고 해도 문제가 되지 않는다. <br><br>
    - [x] 그리고 **클라이언트와 리버스 프록시 사이에만 "Keep-Alive ON"으로 하고, 리버스 프록시와 AP 서버 사이는 Keep-Alive OFF로 한다.** <br><br>
    - [x] 이렇게 하면 AP 서버 측은 프로세스 수가 적더라도 하나의 요청이 종료되면 곧바로 그 후에 다른 요청에 응답할 수 있으며 전체적으로 동시에 다룰 수 있는 클라이언트의 수는 많아지고 또한 전송량도 향상된다. <br><br>
+ **아파치 모듈을 이용한 처리의 제어** <br><br>
    - [x] 리버스 프록시로 아파치를 선택한 경우, 해당 리버스 프록시에 아파치 모듈을 내장해서 HTTP 요청의 전/후처리로 임의의 프로그램을 실행 시킬 수가 있다.  <br><br>
        - **mod_deflate :** 컨텐츠를 gzip 압축하는 아파치 모듈, AP 서버로부터 수신한 HTTP 응답을 클라이언트에 압축해서 보낼 수가 있다. <br><br>
        - **mod_ssl :** AP 서버로부터의 응답을 SSL로 암호화할 수 있다. <br><br>
        - **mod_dosdetector :** DoS 공격 대책용 모듈 <br><br>
    - [x] 이외에도 lighttpd 등, 써드파티 제품 모듈/플러그인을 내장할 수 있는 웹 서버가 몇 종류 있으며, 이를 리버스 프록시로 이용하면 유사한 이점을 얻을 수 있다.

<br>

---

<br>

# 2-2. 캐시서버 도입
> 캐시서버는 인터넷 서비스 속도를 높이기 위해 사용자와 가까운 곳에 데이터를 임시 저장하여 빠르게 제공해주는 Proexy 서버를 의미

<br>

## ⓐ 캐시란
> 데이터에 빠르게 접근하기 위해 자주 사용되는 데이터나 값을 미리 복사해 놓는 임시 장소를 의미한다.

<br>

#### 【 캐시를 적용하기 좋은 데이터 】
> 서버에 데이터를 요청하면, 해당 서버가 응답을 반환할 때까지 페이지가 로드되지 않는다 즉, 페이지 로딩에 필요한 정적 리소스를 캐싱하여 사용하게 되면 요청을 보내는 네트워크 요청 횟수를 줄일 수 있을 뿐만 아니라 서버 응답을 기다려야 할 필요도 없기 때문에 사용자에게 보다 빠르게 화면을 보여줄 수 있다.

<br>

- [x] 자주 참조되는 데이터<br><br>
- [x] 자주 변경되지 않는 데이터<br><br>
- [x] 동일한 입력에 대해 동일한 출력을 보장하는 데이터 <br><br>

#### 【 캐시 종류 】

+ **Private Cache** <br><br>
    - [x] 웹 브라우저에 저장디는 캐시이며, 다른 사람이 접근할 수 없다. <br><br>
    - [x] 단, 서버 응답에 Autorization 헤더가 포함되어 있다면 Private Cache에 저장되지 않는다.<br><br>
+ **Shared Cache** <br><br>
    - [x] Shared Cache는 웹 브라우저와 서버 사이에서 동작하는 캐시를 의미하며, 2가지로 나뉜다.<br><br>
        + **Prxoy Cache :** 포워드 프록시에서 동작하는 캐시 <br><br>
        + **Managed Cache :** CDN 서비스 그리고 리버스 프록시에서 동작하는 캐시

<br>

#### 【 대표적인 오픈소스 캐시 솔루션 】

+ Memcached
+ Redis
+ Squid
+ Nginx / Apache Traffic 서버
+ Varnish Cache

<br>

## ⓑ Squid 캐시서버

<br>

+ Squid는 HTTP, HTTPS, FTP 등에서 이용되는 오픈소스 캐시 Proxy 서버 <br>

+ Squid는 HTTP 프로토콜의 캐시기능을 전제로 한 캐시서버로, 따라서 HTML, CSS, Javascript 등의 정적인 컨텐츠를 캐싱할 때 사용된다.

<br>

#### 【 Squid 설정 예시 】
> Squid를 리버스 프록시로 이용할 경우, 구성은 다양하지만 여기서는 원본 컨텐츠를 지닌 백엔드 AP서버까지 아파치 -> Squid의 두 Reverse Proxy를 경유하게 되는 구성
```
                    192.168.0.150
                <-  [ Squid-1 ]   <-   
 [ AP 서버 ] 	                    		[ Apache ] (mod_proxy_balancer나 LVS로 두 대의 Squid로의 분배를 수행)
192.168.0.100   <-  [ Squid-2 ]   <-	         
                    192.168.0.151
```
```
# Squid를 80번 포트에 바인드
http_port 80 	

# 원본 서버는 백엔드 서버
cache_peer 192.168.0.100 parent 80 no-query originserver

# 형제(sibling) Squid는 192.168.0.151에 있고, 포트 3130으로 송수신함
cache_peer 192.168.0.151 sibling 80 3130                    

# 모든 서버에서 접근가능 (LAN 내부이므로 접근제어를 하지 않음)
http_access allow all						                
			
# 캐시 스토리지는 coss를 이용
cache_dir coss /var/squid/coss 8000 block-size=512 max-size=524288	

# 30분간 컨텐츠를 캐시 
refresh_pattern . 30 20% 3600                                       

# KeepAlive에 의한 접속유지를 무효화
client_persistent_connections off	
server_persistent_connections off

# 형제와의 캐시 존재확인시 타임아웃을 2000ms로 설정
icp_query_timeout 2000			      
```

## ⓒ memcached

+ 동적 컨텐츠의 처리 속도를 높이는데 사용되는 분산 메모리 캐시서버 <br>

+ 서버의 메모리(in-memory)에 저장하는 방식으로, 반환 속도가 빠르고 서버의 성능을 향상하는 것이 목적인 캐시에 최적화 되어 있다. <br>

+ NoSQL 데이터베이스로 key-value 쌍으로 활용해서 데이터를 저장한다. <br>

<br>

---

<br>

# 2-3. MySQL 리플리케이션

<br>

## ⓐ DB 서버가 정지하는 경우	

<br>

+ DB 서버의 프로세스(mysqld)가 비정상 종료함

+ 디스크가 가득참

+ 디스크가 고장 

+ 서버 전원이 고장

<br>

## ⓑ 복구하는 방법

> 동일한 서버를 두 대 준비하는 것은 비교적 간단히 할 수 있지만, 단시간에 복구하는 것을 목표로 함에 있어서 문제가 되는 것은 "DB 데이터"이다 그래서 등작한 것이 "리플리케이션(Replication)"이라는 방법이다.

<br>

#### 【 리플리케이션(Replication) 】
> **데이터를 실시간으로 다른 곳으로 복제하는 것을 말하며 복제를 네트워크를 경유해서 수행하면 물리적으로 다른 서버 간에 데이터를 동일하게 유지할 수 있다.**

<br>

## ⓒ MySQL 리플리케이션 기능의 특징과 주의점

<br>

#### 【 싱글 마스터, 멀티 슬레이브 】

> MySQL 리플리케이션 기능으로 지원되고 있는 것은, 한 대의 마스터와 여러 슬레이브로 이루어진 구성이며, 슬레이브는 여러 대 존재할 수 있으므로 SELECT문 등의 참조 쿼리를 여러 슬레이브로 분산해서 성능 향상을 꾀할 수 있도록 구성할 수도 있다.

<br>

- [x] **마스터 :** 클라이언트로부터 갱신과 참조 두 가지 쿼리를 받아들이는 서버 <br><br>
- [x] **슬레이브 :** 클라이언트로부터 갱신 쿼리는 받아들이지 않으면서 데이터 갱신은 마스터와 연계를 통해서만 수행하는 역할을 하는 서버

<br>

#### 【 비동기 데이터 복사 】

> 비동기란 마스터에 수행한 갱신 처리가 동시에 슬레이브로 반영되지 않는다(반영되기까지 시간차가 있다)는 의미로, 비동기가 아닌 동기 리플리케이션을 지원하는 RDBMS도 있지만, 어느 쪽은 우수하고 다른 쪽은 열등하다고 할 수는 없다.

<br>

## ⓓ 리플리케이션 원리

<br>

#### 【 슬레이브의 I/O 쓰레드와 SQL 쓰레드 】

<br>

- [x] **I/O 쓰레드 :** 마스터에서 얻은 데이터(갱신 로그)를 "릴레이 로그"라고 하는 파일에 단지 기록하기만 한다. <br><br>
- [x] **SQL 쓰레드 :** 릴레이 로그를 읽어서 오로지 실행만 한다. <br>

<br>

#### 【 바이너리 로그와 릴레이 로그 】

+ **바이너리 로그** 

    - [x] **데이터를 갱신하는 처리만이 기록되고 데이터를 참조하는 쿼리는 기록되지 않는다.** <br><br>
    - [x] 바이너리 로그는 리플리케이션 외에도 풀백업에서 갱신된 내용만 보관하고자 할 경우에도 사용된다. <br><br>
    - [x] 바이너리 로그는 텍스트 형식이 아니므로 직접 열어볼 수는 없지만, **mysqlbinlog 명령을 이용해 텍스트 형식으로 변환**할 수 있다.<br><br>

+ **릴레이 로그**

    - [x] **슬레이브의 I/O 쓰레드가 마스터로부터 갱신 로그(갱신 관련 쿼리를 기록한 데이터)를 수신해서 슬레이브 측에 저장한 것**이다. <br><br>
    - [x] 내용은 바이너리 로그와 동일하지만, 바이너리 로그와 달리 필요없어지면 SQL 쓰레드에 의해 자동적으로 삭제된다.<br>

<br>

#### 【 포지션 정보 】
> **슬레이브는 리플리케이션 완료한 위치정보를 알고 있어, 슬레이브의 MySQL를 종료하고 잠시 후에 다시 기동하더라도 종료한 시점부터 데이터 리플리케이션을 재개할 수 있다.** 마스터 호스트명, 로그 파일명, 로그 파일 내에서 처리한 포인트 정보를 "포지션 정보"라 한다. **포지션 정보는 "master.info"라는 택스트 형식의 파일로 관리되고 있으며, "SHOW SLAVE STATUS"라는 SQL문으로 확인할 수 있다.**

<br>

## ⓒ 리플리케이션 구성

<br>

#### 【 리플리케이션 조건 】

+ **마스터는 여러 슬레이브를 가질 수 있다.** 

+ **슬레이브는 마스터를 단 하나만 가질 수 있다.**

+ **모든 마스터, 슬레이브에는 일련의 서버-id를 지정해야 한다.**

+ **마스터는 바이너리 로그를 출력해야 한다.**

<br>

#### 【 my.cnf 】

+ (1) DB 서버마다 개별 값으로 지정할 필요가 있다. (1~4294967295)

+ (2), (3) 바이너리 로그를 활성화하고 바이너리 로그 파일명과 인덱스 파일을 지정

+ (4), (5) 릴레이 로그를 활성화하고 해당 파일명을 지정

+ (6) 슬레이브에서도 바이너리 로그를 출력하도록 지시하기 위한 설정

    - [x] 이 설정이 없으면 슬레이브는 바이너리 로그를 출력하지 않으나, MySQL 리플레키이션에서 마스터는 바이너리 로그를 출력해야 하므로 슬레이브를 마스터로 승격시키는 단계를 부드럽게 진핸하기 위해, 슬레이브에서도 미리 바이너리 로그를 출력하도록 해두는 편이 좋다

```
[mysqld]
server-id       = 1             # (1) 
log-bin         = mysql-bin     # (2) 
log-bin-index   = mysql-bin     # (3) 
relay-log       = relay-bin     # (4)
relay-log-index = relay-bin     # (5)
log-slave-updates               # (6)
```

<br>

#### 【 리플리케이션용 사용자 생성 】
```
GRANT REPLICATION SLAVE ON *.* TO repl@'192.168.219.0/255.255.255.0' INETIFIED BY '1234';
```

<br>

#### 【 리플리케이션 시작시에 필요한 데이터 】
> 슬레이브를 새로 추가한 경우나 고장난 슬레이브를 대채해서 투입할 경우 슬레이브의 초기 데이터는, **마스터의 "풀덤프"만이 아니라 해당 풀덤프가 마스터의 바이너리 로그에서 어느 시점의 것인지 하는 "포지션 정보"도 필요하다.** 

<br>

- [x] 편의상, "풀덤프 + 포지션 정보" 묶음을 여기서는 "스냅샷"이라고 하자 그러면 **스냅샷을 얻기 위해서는 마스터의 mysqld를 정지한 후, MySQL 데이터 디렉터리를 복사하거나 마스터의 바이너리 파일의 이름을 메모**해둔다. <br><br>

- [x] 파일명이 "mysql-bin.000002"인 경우는 mysqld를 실행시에 바이너리 로그는 다음 번호의 로그파일로 변경되므로, 포지션 정보는 "mysql-bin.000003"의 최초가 된다. <br><br>

- [x] mysqld를 정지할 수 없을 경우에는 갱신 관련된 쿼리를 막아놓은 상태로 만든 다음에 풀덤프를 하고, SHOW MASTER STATUS의 결과를 메모해두면 된다.

<br>

## ⓓ 리플리케이션 시작

<br>

#### 【 Master 서버 - DB 생성 】
```
mysql> create database repl_db default character set utf8;
```

<br>

#### 【 Slave 서버 】

+ **my.cnf 설정**

    - [x] 복제하고자 하는 데이터베이스를 의미하며 2개이상의 데이터베이스를 할경우 replicate-do-db를 추가로 작성하면 된다.

```
서버-id=2
replicate-do-db='repl_db'
```

<br>

+ **Master 서버로 연결하기 위한 설정**

```
change master to
master_host='192.168.219.148',
master_user='repl',
master_password='1234',
master_log_file='mysql-bin.000010',
master_log_pos=1487;

SLAVE START
```

<br>

+ **my.cnf 파일에 Slave 서버 설정 후 재시작**

```
[mysqld]
replicate-do-db='repl_db'
master-host=192.168.219.148
master-user=repl
master-password=1234
master-port=3306
server-id=2
```
```
service mysqld restart
```

<br>

## ⓔ 리플리케이션 상황 확인

<br>

#### 【 마스터 - 마스터의 바이너리 로그 상황을 확인 】
```
SHOW MASTER STATUS\G;
```

<br>

#### 【 마스터 - 현재 마스터에 존재하는 모든 바이너리 로그의 파일명을 출력 】
```
SHOW MASTER LOGS;
```

<br>

#### 【 마스터 - 특정 바이너리 로그를 제외한 나머지 삭제 】
```
PURGE MASTER LOGS TO 'mysql-bin-000003'
```

<br>

#### 【 슬레이브 - 슬레이브의 정보를 확인 】
```
SHOW SLAVE STATUS\G;
```

<br>

---

<br>

# 2-4. MySQL 슬레이브 활용

<br>

## ⓐ 여러 슬레이브에 분산하는 방법

<br>

+ 애플리케이션으로 분산하기 (O/R Mapper를 통해 DB에 접근)

    - [x] 슬레이브군의 호스트명 목록을 얻는다.

    - [x] 분산할 슬레이브 서버를 결정하는 로직을 구현한다.
    
    - [x] 슬레이브의 헬스체크를 수행하고, 다운된 슬레이브에는 분산되지 않도록 하는 처리를 구현한다.

<br>
    
+ 내부 로드밸런서로 분산하기 (애플리케이션으로 분산하기와의 비교)

    - [x] 애플리케이션은 슬레이브의 대수를 신경쓰지 않아도 된다.

    - [x] 애플리케이션은 슬레이브의 상태를 신경쓰지 않아도 된다.
    
    - [x] 보다 균일한 분산을 할 수 있다.


<br>

## ⓑ 슬레이브 참조를 로드밸런서 경유로 수행하는 방법

![내부 로드밸런서 경유 슬레이브 참조](https://user-images.githubusercontent.com/42735894/218947614-eb23a6c3-c9b0-4e43-a72e-644acdcdd2e6.PNG)

```
keepalived.conf

# Basic Section
vrrp_instance VI {
    state               BACKUP
    interface           eth0
    garp_master_delay   5
    virtual_router_id   230                 # (1)
    priority            100
    advert_int          1
    authentication {
        auth_type   PASS
        auth_pass   himitsu
    }
    virtual_ipaddress {
        192.168.219.230/24  dev eth0        # ┐
        192.168.219.119/24  dev eth0        # ┘ (2)
    }
}

# MySQL slave section
virtual_server_group MYSQL100 {
    192.168.219.119 3306
}
virtual_server group MYSQL100 {
    delay_loop  3
    lvs_sched   rr
    lvs_method  DR                          # (3)
    protocol    TCP

    real_server 192.168.219.111 3306 {
        weight  1
        inhibit_on_failure
        TCK_CHECK {
            connect_port    3306
            connect_timeout 3
        }
    }
    real_server 192.168.219.112 3306 {
        weight  1
        inhibit_on_failure
        TCK_CHECK {
            connect_port    3306
            connect_timeout 3
        }
    }
}
```

<br>

+ (1) VRID(VRRP 라우터 그룹 식별자)를 지정, VRRP에서는 VRID가 같은 노드의 그룹으로 가상 라우터를 구성한다 따라서 동일한 네트워크 세그먼트에서는 가상 라우터 그룹마다 다른 VRID를 붙여야 한다. 만일 외부 로드밸런서와 같이 이미 가상 라우터 그룹이 존재하고 있는 경우 중복되지 않도록 해야한다 (tcpdump 명령으로 VRRP 패킷, VRID를 볼 수 있음)

<BR>

+ (2) 내부 로드밸런서 자신의 가상 라우터 주소(192.168.219.230), 가상 슬레이브(db100-s)용 주소(192.168.219.119)

<BR>

+ (3) DSR 분산하도록 설정했으므로, 가상 슬레이브의 IP 주소별, 패킷을 받아들이도록 해야 한다. (슬레이브 각각(db101, db102)에 설정)

    ```
    iptables -t nat - A PREROUTING -d 192.168.219.119 -j REDIRECT
    ```

+ 테스트

    ```
    $ check_lb_slave() {
        echo 'SHOW VARIABLES LIKE "server_id"' | mysql -s -hdb100-s
    }

    $ check_lb_slave
    server_id       101
    $ check_lb_slave
    server_id       102
    ```

<br>

## ⓒ 내부 로드밸런서의 주의점 (DSR)

> 내부 로드밸런서의 주의사항은 **분산방법은 NAT가 아닌 DSR로 하는 것**이다.

<br>

- [x] NAT의 경우, 클라이언트가 VIP 목적지 주소로 요청을 보내면, 로드밸런서는 목적지 주소를 Real 서버로의 주소로 수정해서 Real 서버에 전송한다 응답할 때도 마찬가지로
    로드밸런서를 경유하면 그 때 출발지 주소가 VIP로 변경되므로 문제가 일어나지 않는다. <br><br>

- [x] 하지만 Real 서버와 클라이언트가 같은 네트워크에 존재할 때는, 로드밸런서를 도입할 필요없이 직접 클라이언트로 패킷이 전송되어, 결과적으로 VIP 앞으로 전송된 패킷의 응답이 VIP와는 다른 곳(Real 서버의 주소)으로부터 반환되어 오는 것처럼 보이는 것이다. <br><br>

<br>

---

<br>

# 2-5. 고속, 경량의 스토리지 서버 선택

<br>

## ⓐ 스토리지 서버의 필요성
> 부하분산 환경에서는 **여러 대의 웹 서버에 동일한 파일을 저장하고 파일의 수나 크기가 방대해지면 다음과 같은 문제가 발생**한다.

<br>

+ **모든 웹 서버로 배치(Deploy)시키는는 데에는 시간이 걸린다.**

+ **모든 웹 서버에 대용량의 하드디스크를 탑재하야 한다.**

+ **모든 웹 서버의 파일이 정합성(데이터가 서로 모순 없이 일관되게 일치해야 함)을 갖는지 검증하기가 곤란하다.**

+ **웹 서버를 신규로 도입하기가 곤란하다(파일 복사에 시간이 걸린다).**

<br>

## ⓑ 시스템 관리자 입장에서의 문제 

<br>

#### 【 스토리지 서버는 단일장애지점(SPOF, Single Point Of Failure)이 되기 쉽다. 】

> 웹 서버가 스토리지 서버를 NFS 마운트해서 이용하는 경우, **스토리지 서버가 정지하고 있는 동안 웹 서버는 NFS로 파일 작업을 끊임없이 재시도하게 되는데, 그 결과 웹 서버는 파일 작업 대기 프로세스로 가득 차서 다른 페이지도 조회할 수 없는 상태(서비스 정지)에 빠져버리게 된다.** <br><br>
마운트 옵션으로 soft와 intr을 지정함으로써 어느 정도 개선은 할 수 있지만, **NFS는 운영체제의 기능으로서 구현되어 있으므로 Web 애플리케이션에서 타임아웃 시간을 조정하거나 파일 작업을 중단할 수는 없다 따라서 파일 작업이 타임아웃되기를 기다리는 동안 웹 서버의 프로세스가 가득 차게 되면 서비스는 정지해버린다.**

<br>

+ man nfs - mount 옵션 soft, hard, intr 참고 (https://linux.die.net/man/5/nfs)

<br>

#### 【 스토리지 서버는 병목이 되기 쉽다. 】

![222](https://user-images.githubusercontent.com/42735894/219003844-3547d1f0-5f59-4bf8-abd7-ea6cf5de3214.png)

+ 웹 서버는 10~20대로 확장할 수 있지만, NFS 서버는 확장할 수 없다 이로 인해 여기서 병목이 돼버리면 이를 개선하기는 상당히 곤란하다.

+ NFS 서버를 증설해서 디렉터리를 나눠서 처리하는 방법도 있지만, 이것만으로는 해결할 수 없느 경우가 많다.

+ 즉, 접속이 집중되고 있는 상황에는 대다수의 사용자가 동일한 데이터를 요청하고 있고, 디렉터리별로 NFS서버를 분할했다고 해도 결국은 동일한 NFS 서버에 대해 접속이 집중된다.

<br>

#### 【 여러 대의 서버에 파일 동기화 문제 】

![1313ew](https://user-images.githubusercontent.com/42735894/219005505-d2806a8a-5f42-4d8e-9112-d3d5c9596d46.PNG)

+ 부하 문제만을 고려한다면 웹 서버마다 마운트할 NFS 서버를 나누면 대응은 가능하지만, 이런 구성을 하면 NFS 서버에서 파일의 정합성을 유지해야 한다는 문제가 남는다.

+ 컨텐츠를 배치할 때는 모든 NFS 서버에 동일한 파일을 전송해야 하지만, 파일의 개수가 많아지면 모든 서버에 내용이 동일한지 여부를 체크하기는 어려워진다.

<br>

## ⓒ 이상적인 스토리지 서버

<br>

+ **대량 접속에도 병목되지 않을 정도로 빠른 스토리지 서버**

+ **여러 대의 서버에 파일을 동기화하는 것을 피해야 함**

+ **단일장애지점(SPOF)이 되지 않아야 함**

<br>

#### 【 대량 접속에도 병목되지 않을 정도로 빠른 스토리지 서버 - HTTP를 스토리지 프로토콜로 이용 】

> **접속이 집중될 때라는 것은 수많은 사용자가 동일한 데이터를 요청하고 있을 때** 즉, 동일한 데이터를 반복해서 읽어내는 형태의 접속 패턴이 된다. 많은 **웹 사이트에서 스토리지 서버에 요구하는 것은 「읽기 속도」와 「디스크 용량」일 것이며, 의외로 「쓰기」에 대한 성능을 요구되지 않는다.** <br><br>
> **고속 쓰기가 요구되는 데이터는 「세션 정보」와 「개인 정보」 등이 대부분을 차지하고 있는데 「세션 정보」의 경우는 일시적인 데이터이므로 memcached 등의 메모리 기반 캐시서버를 사용하면 되고, 「개인 정보」는 DB에 저장하면 되는 데이터**이다.<br><br>
> **즉, 스토리지 서버는 동영상, 이미지 등 크기가 큰 데이터를 가능한 많이 저장할 수 있고 필요한 것을 고속으로 읽어낼 수 있으면 되는 것**이다. 스토리지 서버라고 해도 NFS 등을 반드시 이용해야만 하는 것은 아니며 웹 애플리케이션에서 자주 사용되는 플랫폼 PHP, JSP 등에서는 웹 서버상의 파일을 일반 파일과 동일하게 다룰 수 있으므로, **HTTP 경유하여 파일을 취득할 수 있다.**

![313213213](https://user-images.githubusercontent.com/42735894/219027303-65ff8edd-dc58-412b-b36d-1b01df3096d2.PNG)

+ WS 에서 사용하는 웹 서버는 동적 페이지를 생성 기능은 불필요하며 얼마나 고속으로 정적인 파일 전송할 수 있는지가 중요한데 이때 「lightppd, nginx 등의 경량 웹 서버」를 이용해 HTTP를 지원함으로 성능을 향상시킬 수 있다.

+ 이러한 데이터는 거의가 스토리지 서버의 메모리에 캐싱되며, lightppd, nginx 등의 경량 웹 서버는 메모리에 캐싱되어 있는 데이터를 오로지 전송하기만 하면 되므로 스토리지 서버의 디스크 I/O에는 거의 부하가 걸리지 않는다 따라서 스토리지 서버가 병목이 될 가능성은 없다.

+ NFS에 비하면 HTTP는 서버와 클라이언트의 결합이 느슨하다고 할 수 있다.

    - [x] 웹 서버가 NFS 마운트하고 있을 경우, NFS 서버가 정지하면 파일시스템 레벨에서 처리를 정지해버리기 때문에 아파치를 재실행하는 것도 뜩대로 되지 않는 상황에 빠지게 된다.

    - [x] 그러나, HTTP라면 웹 애플리케이션 측에서 자유롭게 타임아웃을 설정할 수 있으므로 스토리지 서버의 이상을 검출해서 간단하게 에러 메시지를 반환할 수 있게 된다 따라서 스토리지 서버의 장애에 의해 사이트가 모두 정지해버릴 위험성을 다소 피할 수 있다.

<br>

#### 【 "여러 대의 서버에 파일을 동기화하는 것을 피해야 함"과 "단일장애지점(SPOF)이 되지 않아야 함"을 해결할 방법 】

<br>

+ [3-2. 스토리지 서버의 다중화 - DRBD로 미러링 구성](https://github.com/revenge1005/System-Redundancy/tree/master/ch.03#3-2-%EC%8A%A4%ED%86%A0%EB%A6%AC%EC%A7%80-%EC%84%9C%EB%B2%84%EC%9D%98-%EB%8B%A4%EC%A4%91%ED%99%94-%EB%AF%B8%EB%9F%AC%EB%A7%81-%EA%B5%AC%EC%84%B1)