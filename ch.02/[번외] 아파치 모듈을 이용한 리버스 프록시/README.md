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

+ 대량의 연결이 필요한 서비스에서 사용을 권장

<br>

#### 【 Event (Default 설정) 】

+ Apache 2.4 버전부터 지원되는 방식으로, worker 방식을 기반으로 한다.

+ 기존에는 클라이언트의 연결이 완전히 끝나지 않는 한 하나의 프로세스를 계속 유지해야 했다. (Keepalive)

+ **Event 방식은 전용 리스너 쓰레드를 생성하여 처리하도록 하며 요청을 받아들이는 쓰레드와 요청을 처리하는 쓰레드를 분리하여 몰려오는 요청을 분산처리한다.**

<br>

## ⓑ worker 방식 설정 
> 더 자세한 내용은 참고 - https://httpd.apache.org/docs/2.4/mod/worker.html

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
    ThreadsPerChild		128	# 변경
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

+ 특정 웹 서버상에서의 최대 프로세스/쓰레드 수는 리소스가 상한선에 이르렀을때 스왑이 발생하지 않을 정도 즉,

    - [x] **OS나 웹 서버 이외의 소프트웨어가 항시 이용하는 메모리량**

    - [x] **웹 서버의 프로세스/쓰레드 수가 최대수에 이르렀을 때 서버가 소비하는 합계 메모리량**


+ 위 두 가지를 합해서 탑재된 물리적 메모리의 범위 내에 들 정도로 튜닝하는 것이 적합하다.

+ 더 자세한 내용은 [4. 프로세스/쓰레드 당 메모리 사용량의 판별방법]()