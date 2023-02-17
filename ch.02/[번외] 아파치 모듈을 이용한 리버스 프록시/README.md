## 목차

ⓐ [Apache MPM(Multi Processing Module)](#ⓐ-apache-mpmmulti-processing-module)

ⓑ [worker 방식 설정](#ⓑ-worker-방식-설정)

ⓒ [최대 프로세스/쓰레드 수 설정](#ⓒ-최대-프로세스쓰레드-수-설정)

ⓓ [ServerLimit/ThreadLimit와 메모리의 관계](#ⓓ-serverlimitthreadlimit와-메모리의-관계)

ⓔ [Keep-Alive 설정](#ⓔ-keep-alive-설정)

ⓕ [필요한 모듈 로드](#ⓕ-필요한-모듈-로드)

ⓖ [Rewriterule 설정](#ⓖ-rewriterule-설정)

ⓗ [mod_proxy_balancer로 여러 호스트로 분산하기](#ⓗ-mod_proxy_balancer로-여러-호스트로-분산하기)

<br>

---

<br>

# 번외-1. 아파치 모듈을 이용한 리버스 프록시

<br>

## ⓐ Apache MPM(Multi Processing Module)
> 여러 이용자가 동시에 웹 서버에 요청하였을 때 이 요청들을 처리하기 위해 자식 프로세스들에게 분배하는 모듈

<br>

#### 【 Prefork (1 Process - 1 Thread 방식) 】

+ **하나의 요청에 하나의 웹 서버 프로세스를 할당하여 처리하도록 하는 방식**

+ 요청이 독립적인 프로세스로 처리되기 때문에 프로세스 오류가 발생해도 다른 요청에 영향이 없다.

+ 프로세스가 독립적으로 실행되므로 프로세스 수가 많아짐에 따라 시스템 리소스에 부하가 심해진다.

+ 요청이 많이 없고 각각의 연결 안정성이 중요한 서비스에 사용을 권장한다.

<br>

#### 【 worker (1 Process - Multi Thread 방식) 】

+ **요청을 쓰레드 단위로 처리하는 방식**

+ 연결마다 메모리 공간을 공유하여 리소스 부하가 줄어든다는 장점이 있지만, 오류 발생시 한 프로세스 내 모든 쓰레드들이 영향을 받을 수 있다.

+ 대량의 연결이 필요한 서비스에서 사용을 권장한다.

<br>

#### 【 Event (Default 설정) 】

+ Apache 2.4 버전부터 지원되는 방식으로, worker 방식을 기반으로 한다.

+ 기존에는 클라이언트의 연결이 완전히 끝나지 않는 한 하나의 프로세스를 계속 유지해야 했다. (Keepalive)

+ **Event 방식은 전용 리스너 쓰레드를 생성하여 처리하도록 하며 요청을 받아들이는 쓰레드와 요청을 처리하는 쓰레드를 분리하여 몰려오는 요청을 분산처리한다.**

<br>

## ⓑ worker 방식 설정 
> 더 자세한 내용은 -> https://httpd.apache.org/docs/2.4/mod/worker.html

```
<IfModule mpm_worker_module>
		StartServers		2
		MaxClients		    150
		MinSpareThreads		25
		MaxSpareThreads		75
		ThreadsPerChild		25
		MaxRequestPerChild	0
</IfModule>
```

<br>

+ **『StartServers』**

    - [x] **시작시에 생성되는 서버 프로세스의 개수**

    - [x] 자식 프로세스의 수는 부하에 따라 동적으로 변경되기 때문에 이 값은 큰 의미가 없습니다.

<br>

+ **『MaxClients』**

    - [x] **동시에 처리될 최대 커넥션(request)의 수**

    - [x] MaxClients 수치를 초과한 후 온 요청들은 ListenBackLog에 의해 대기상태가 됩니다.

<br>

+ **『MinSpareThreads』**

    - [x] **최소 쓰레드 개수**

    - [x] **서버에 idle 쓰레드가 충분하지 않다면 child 프로세스는 idle 쓰레드가 MinSpareThreads 보다 커질때까지 생성됩니다.**

<br>

+ **『MaxSpareThreads』**

    - [x] **최대 쓰레드 개수**

    - [x] **서버에 너무 많은 idle 쓰레드가 존재하면 child 프로세스는 idle 쓰레드가 MaxSpareThreads 수보다 작아질 때까지 kill 됩니다.**

<br>

+ **『ThreadPerChild』**

    - [x] **개별 자식 프로세스가 지속적으로 가질 수 있는 Thread의 개수**
    

<br>

+ **『MaxRequestPerChild』**

    - [x] **자식 프로세스가 서비스할 수 있는 최대 요청 개수**

<br>

## ⓒ 최대 프로세스/쓰레드 수 설정
> **위 설정 중에 중요한 지표는 "MaxClients"와 "ThreadPerChild"이다.** <br><br> worker 모델의 경우, 아파치는 여러 자식 프로세스를 실행시키고 각각의 프로세스 내에 여러 쓰레드를 생성해서 결국 『**프로세스 수 X 프로세스 당 쓰레드 수**』만큼의 요청을 동시에 처리할 수 있게 된다. <br><br> **여기서 프로세스 당 쓰레드 수를 제어하는 것이 바로 "ThreadPerChild"이며, 자식 프로세스의 최대 값은 "MaxClient / ThreadPerChild"으로 결정된다. MaxCliens는 동시에 처리할 수 있는 클라이언트의 총수가 된다.** <br><br> 따라서 앞에서 본 설정의 경우는 다음과 같이 설정된다. 

<br>

+ **"최대 프로세스 수" :** 6 <br><br>
+ **"프로세스 당 최대 쓰레드 수" :** 25 <br><br>
+ **"동시에 처리할 수 있는 클라이언트 수" :** 6 X 25 = 150

<br>

## ⓓ ServerLimit/ThreadLimit와 메모리의 관계
```
<IfModule mpm_worker_module>
    StartServers		2
    ServerLimit		    32	    # 신규
    ThreadLimit		    128 	# 신규	
    MaxClients		    4096	# 변경
    MinSpareThreads		25
    MaxSpareThreads		75
    ThreadsPerChild		128	    # 변경
    MaxRequestPerChild	0
</IfModule>
```

<br>

#### 【 ServerLimit/ThreadLimit 】

+ **ServerLimit/ThreadLimit은 MaxClients나 ThreadsPerChild와 병행해서 프로세스/쓰레드의 최대 생성수를 결정하는 또 다른 설정항목**이다.

+ **ServerLimit은 디폴트로 16, ThreadLimitd은 64로 되어 있어 그 이상의 수를 설정할 경우에는 이 두 항목을 명시적으로 지정할 필요가 있다.**

<br>

#### 【 ServerLimit/ThreadLimit과 MaxClients/ThreadsPerChild의 차이 】

+ **MaxClients/ThreadsPerChild는 서버의 동적인 리소스 소비에 관련된 설정항목으로, 이 값이 높든 낮든 서버가 최소한으로 소비하는 리소스 소비량에는 영향을 미치지 않는다.**

+ **ServerLimit/ThreadLimit는 아파치가 확보하는 공유 메모리의 크기에 영향을 주며, 이 값에 필요 이상으로 높은 값을 설정하면 그만큼 아파치는 쓸데없이 공유 메모리를 소모하게 되는 것이다.**

+ 따라서 설정하고자 하는 프로세스/쓰레드 수의 상한선이 내장된 값인 "ServerLimit 16, ThreadLimitd 64"를 초과할 경우에만 그 값에 맞게 설정된다.

<br>

#### 【 프로세스/쓰레드당 메모리 사용량에 대해 ... 】

+ **프로세스/쓰레드당 메모리 사용량은 내장된 모듈의 종류에 의존하며 또한 OS가 아파치 메모리를 얼마나 할당할지는 환경에 따라 다르므로 설정값을 단언할 수는 없다.**

+ 특정 웹 서버상에서의 최대 프로세스/쓰레드 수는 리소스가 상한선에 이르렀을때 스왑이 발생하지 않을 정도 즉, <br><br>

    - **OS나 웹 서버 이외의 소프트웨어가 항시 이용하는 메모리량** <br><br>

    - **웹 서버의 프로세스/쓰레드 수가 최대수에 이르렀을 때 서버가 소비하는 합계 메모리량**

<br>

+ 위 두 가지를 합해서 탑재된 물리적 메모리의 범위 내에 들 정도로 튜닝하는 것이 적합하다.

+ 더 자세한 내용은 [4. 프로세스/쓰레드 당 메모리 사용량의 판별방법]()

<br>

## ⓔ Keep-Alive 설정

<br>

#### 【 Kepp-Alive 】

+ 최초 요청 시 연결된 서버와의 접속을 해당 요청이 종료한 후에도 접속을 끊지 않고 유지한 채로 이어지는 요청에 해당 접속을 계속 사용함으로써, 하나의 접속으로 다수의 요청을 처리할 수 있도록 실현한 것

<br>

#### 【 옵션 】
```
# Keep-Alive 옵션 활성화
KeepAlive On

# Keep-Alive 상태로 처리할 수 있는 최대 요청수 지정 (100건)
MaxKeepAliveRequests 100

# Keep-Alive 타임아웃(클라이언트와 접속을 유지하는 시간)을 지정 (5초)
KeepAliveTimeout 5   
```

+ KeepAliveTimeout 값을 크게 하면 그만큼 프로세스/쓰레드가 Keep-Alive를 위해 점유하는 시간이 길어지므로, 서버의 리소스 소비량이 커진다.

<br>

## ⓕ 필요한 모듈 로드

<br>

#### 【 최소한으로 필요한 모듈은 다음과 같다. 】

+ **mod_rewrite** 

+ **mod_proxy**

+ **mod_proxy_http**

+ **mod_alias** # 없어도 되지만 있으면 편리함 (공개 디렉터리의 앨리어스를 정의할 수 있음)

<br>

#### 【 모듈 로드 】
```
LoadModule alias_module modules/mod_alias.so
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

<br>

## ⓖ RewriteRule 설정
> 참조 (1) - https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html#rewriterule <br>
> 참조 (2) - https://zetawiki.com/wiki/%EC%95%84%ED%8C%8C%EC%B9%98_mod_rewrite_RewriteRule_%ED%94%8C%EB%9E%98%EA%B7%B8

<br>

#### 【 설정 조건 】

+ /images는 이미지를 전송하는 경로로, 이 URL은 리버스 프록시에서 직접 전송. 한편, 모든 이미지는 리버스 프록시와 동일한 호스트 내의 /path/to/images/ 아래에 두기로 한다. (/css, /js 도 마찬기지)

+ 그 밖의 URL은 동적 컨텐츠 전송. AP 서버 192.168.0.100으로 요청을 프록시한다.

<br>

#### 【 RewriteRule 설정 】
```
Listen 80
<VirtualHost *:80>
	ServerName proxy.test.co.ko
		
	Alias	/images/	"/path/to/images/"
	Alias	/css/	    "/path/to/css/"
	Alias	/js/	    "/path/to/js/"
	
	RewriteEngine	on
	RewriteRule	^/(images|css|js)/ - [L]                # (1)
    RewriteRule	^/(.*)$ http://192.168.0.100/$1 [P,L]   # (2)
</VirtualHost>
```

+ (1) : /images/, /css/, /js/ 중 하나의 URL이 매치될 경우, 별다른 처리 없이 ([L]은 RewriteRule의 패턴매치를 여기서 마친다는 의미), 디폴트 컨텐츠 핸들러로 컨텐츠를 반환한다. 

+ (1) : 클라이언트로부터의 요청이 "/images/profile/test.png" 인 경우, 로컬의 "/path/to/images/profile/test.png"가 클라이언트로 반환된다.

+ (2) : [P] - Proxy

<br>

#### 【 특정 호스트로부터 요청 금지 】
```
RewriteCond %{REMOTE_ADDR} ^192\.168\.0\.200$
RewriteCond .* - [F,L]
```

+ 192.168.0.200으로부터의 요청에 대해  403을 반환하고 종료

+ AP 서버로 프록시하기 전에 조건판정을 하는 RewriteCond 지시문 

<br>

#### 【 로봇으로부터의 요청에 대해 캐시서버 경유 】
+ Squid로 구축한 HTTP 캐시서버가 192.168.0.150에 위치해 있다고 하자, 로봇으로부터의 요청 즉 특정 User-Agent로부터의 요청만 캐싱된 내용을 반환하고자 할 경우는 다음과 같이 설정한다.

+ mod_setenvif를 사용해서 User-Agent 문자열로부터 로봇인지 판정해서 로봇으로 판정된 경우에는 캐시서버로 프록시한다.

```
# SetEnvIf Directive를 유효하게 하기 위해 mod_setenvif를 로드
LoadModule setenvif_module modules/mod_setenvif.so

# User-Agent에 "Yahoo! Slurp" 혹은 "Googlebot"이 포함된 경우 환경변수 ISRobot을 참으로 설정
SetEnvIf User-Agent "Yahoo! Slurp" IsRobot
SetEnvIf User-Agent "Googlebot" IsRobot

RewriteEngine on
	
# 환경변수 IsRobot이 참인 경우 캐시서버로 프록시
RewriteCond %{ENV:IsRobot} .+
RewriteCond ^/(.*)$ http://192.168.0.150/$1 [P,L]

# 그 밖에는 통상적으로 프록시
RewriteRule	^/(images|css|js)/ - [L]
RewriteRule	^/(.*)$ http://192.168.0.100/$1 [P,L]
```

<br>

## ⓗ mod_proxy_balancer로 여러 호스트로 분산하기

<br>

#### 【 AP 서버가 여러 대인 경우의 구성 】

+ "**mod_proxy_balancer를 이용해서 하나의 리버스 프록시로부터 여러 AP 서버로 분산**"

    - [x] mod_proxy_balancer는 리버스 프록시를 수행함에 있어서 프록시 될 호스트가 여러 대 있는 경우라도 각 호스트로 요청을 분산해서 할당할 수 있도록 처리해주는 모듈

    - [x] 또한, 프록시될 호스트가 특정 이유로 응답할 수 없는 경우 장애극복을 해서 분산할 호스트 목록에서 해당 호스트를 분리하고, 해당 호스트가 요청 가능해졌을 때 Failback하는 기능도 가지고 있다.

<br>

+ "**리버스 프록시와 AP 서버 사이에 LVS를 넣는다.**"

    - [x] 리버스 프록시와 AP 사이에 LVS+keepalived를 넣은 구성으로 가장 확실한 방법이라 할 수 있다.

    - [x] mod_proxy_balancer의 장애극복 기능은 LVS의 비해 신뢰성은 그다지 높지는 않다고 생각하며, LVS+keepalived는 mod_proxy_balancer에 비해 부하분산 로직을 조정하기 쉽고 관리하기 편리하다.

    - [x] 다만, LVS+keepalived는 존비하기에 수고스럽고 추가적인 서버가 필요하다
    
<br>

#### 【 mod_proxy_balancer 설정 예시 】
```
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so

<Proxy balancer://backend>
    BalancerMember http://192.168.0.100 loadfactor=10
	BalancerMember http://192.168.0.101 loadfactor=10
	BalancerMember http://192.168.0.102 loadfactor=10
</Proxy>


Listen 80
<VirtualHost *:80>
	ServerName proxy.test.co.ko

	Alias	/images/	 "/path/to/images/"
	Alias	/css/	 "/path/to/css/"
	Alias	/js/	 "/path/to/js/"
	
	RewriteEngine	on
	RewriteRule	^/(images|css|js)/ - [L]
	RewriteRule	^/(.*)$ balancer://backend/$1 [P,L]
</VirtualHost>
```