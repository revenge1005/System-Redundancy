## 목차

1. [다중화](https://github.com/revenge1005/System-Redundancy/tree/master/ch.01)

2. [리버스 프록시, 캐시서버, MySQL 리플리케이션](https://github.com/revenge1005/System-Redundancy/tree/master/ch.02)

3. [DNS 서버의 다중화, 스토리지 서버의 다중화, 네트워크의 다중화, VLAN 도입](https://github.com/revenge1005/System-Redundancy/tree/master/ch.03)

99. [용어 정리](#용어-정리)

99. [참고 자료](#참고-자료)
 

<br>

---

<br>

# 용어 정리

<table>
<tr>
<th align="center">
<img width="441" height="1">
<p> 
<small>
항목 
</small>
</p>
</th>
<th align="center">
<img width="441" height="1">
<p> 
<small>
설명
</small>
</p>
</th>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
AP서버(Application Server)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
애플리케이션 서버, 동적 컨텐츠를 반환하는 서버를 의미함
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
CDN(Content Delivery Network)
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
컨텐츠를 전송하기 위한 네트워크 시스템으로 전송 성능향상과 가용성 향상을 목적으로하며, 전 세계에 존재하는 캐시 서버 중에 클라이언트에 보다 가까운 캐시 서버를 선택해서 전송함으로써 성능향상을 실현하는 것이 구성상 특징임
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
IPVS(IP Virtual Server)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
LVS(Linux Virtual Server) 프로젝트의 결과물로, 로드밸런서에 불가결한 "부하분산" 기능을 실현함
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
LVS(linux Virtual Server)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
리눅스에서 확장성이 있고 가용성이 높은 시스템을 만드는 것을 목표로 하고 있는 프로젝트, 그 성과물 중 하나로 리눅스 로드밸런서를 위한 IPVS가 있다
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
Netfilter
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
리눅스 커널 상에서 네트워크 패킷을 조작하기 위한 프레임워크, 패킷 필터링 등을 수행하는 iptables나 로드밸런서를 실현하기 위한 IPVS도 Netfilter의 기능을 이용하고 있다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
VIP(Virtual IP Address)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
물리적인 서버의 NIC가 아니라 유동적인 서비스나 역할에 할당된 IP 주소를 말함(가상 주소라고도 함). 예를 들면, 로드밸런서의 경우에는 클라이언트의 요청을 받아주는 IP주소를 VIP라고 하는데 이 IP 주소는 HTTP 등의 서비스에 관련된 것이기 때문이며 또한 다중화를 위해 Active/Backup 구성을 할 경우에는 유일한 마스터가 되는 Active측의 로드밸런서가 이 IP 주소를 인계하기 때문이다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
가용성
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
시스템을 정지시키지 않음을 뜻하고 "가용성이 높다"라고 하면 해당 서비스는 거의 멈추지 않는다라는 의미다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
확장성
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
이용자나 규모가 증대됨에 따라 시스템을 확장해서 대응할 수 있는 능력의 정도를 나타냄
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
다중화
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
시스템의 구성요소를 여러 개 배치해서 하나가 고장 나서 정지해도 바로 교체해서 서비스가 멈추지 않도록 하는 것을 말함
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
네트워크 부트(Network Boot)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
네트워크를 통해 부팅에 필요한 부트로더나 커널 이미지 등을 전달받아 기동하는 것, PXE는 네트워크 부트를 실현하기 위한 사용 중 하나이다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
단일장애지점(Single Point of Failure)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
장애가 발생하면 시스템 전체가 정지해버리는 곳을 의미함
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
데몬(Daemon)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
백그라운드에서 지속적으로 실행되면서 특정 작업을 수행하는 프로그램
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
데이터센터(Data Center)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
서버 등의 기기를 수용하기 위해 만들어진 전용시설의 명칭으로 공조, 정전대책, 소화, 지진대책 같이 24시간 365일 서비스를 수행하기 위해 필요한 설비가 갖춰져 있다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
로드밸런서
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
클라이언트와 서버 사이에 위치해서 클라이언트로부터의 요청을 백엔드(Backend)의 여러 서버로 적절하게 분산하는 역할을 하는 장치
<br><br>
다르게 표현하면, 여러 서버를 묶어서 하나의 고성능 가상서버에 준하는 성능을 내기 위한 장치라고도 한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
리소스
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
CPU나 메모리, 하드디스크 등, 서버가 지닌 하드웨어적인 자원
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
메모리 파일시스템(Memory File System)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
하드디스크와 같은 영구기역장치가 아닌 메모리상에 만든 파일시스템
<br><br>
디스크상의 파일시스템과 동일하게 사용할 수 있으나 메모리상에 있기 떄문에 재부팅하면 데이터가 사라지는 반면, 읽고 쓰기를 고속으로 수행할 수 있다는 장점이 있다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
부하(Load)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
부하는 여러 종류가 있는데 크게 "CPU 부하"와 "I/O 부하"로 나눌 수 있다
<br><br>
부하를 계산하기 위한 지표는 Load Average 등 몇 가지가 있으며, 부하를 계측하기 위한 명령어도 top, vmstat 등 몇 가지가 있다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
병목(Bottleneck)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
시스템 전체의 성능을 떨어뜨리는 원인이 되는 지점
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
블록되다(Blocked)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
읽기 또는 쓰기처리가 완료되기를 기다리기 위해 다른 처리를 할 수 없는 상태를 "I/O 대기로 블록되어 있다"라고 한다.
<br><br>
주로 디스크 I/O나 네트워크 I/O에 대해 사용되는 용어지만 입출력 처리 일반에서도 사용되기도 한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
서버팜(Server farm)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
수많은 서버가 모여서 구성된 인프라 시스템, 문맥에 따라서는 데이터센터와 같은 시설을 나타내는 의미로 사용
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
스케일 아웃(Scale-out)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
서버를 여러 대 두고 분삼함으로써 시스템 전체의 성능을 향상시키는 것
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
스케일 업(Scale-up)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
단일 서버의 성능을 높임으로써 시스템 전체의 성능을 향상시키는 것
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
스테이징 환경(Staging Environment)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
실 서비스에 투입하기 전에 최종적인 동작을 확인하기 위한 환경
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
프로덕션 환경(Production Environment)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
실제 서비스를 하고 있는 환경
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
장애극복(Failover)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
다중화된 시스템에서 Active인 노드가 정지했을 때 자동적으로 Backup 노드로 전환하는 것
<br><br>
장애극복(Failover)는 자동, 수동으로 전환되는 것을 스위치오버(Switchover)라고 함
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
페일백(Failback)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
Active 노드가 정지한 후 장애극복된 상태에서 원래의 정상상태로 복귀하는 것
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
전송량(Throughput)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
네트워크와 같은 데이터 통신 측면에서 사용할 경우, 단위시간당 데이터 전송량을 의미
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
지연시간(Latency)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
네트워크와 같이 데이터 통신 특면에서 사용할 경우, 데이터가 도달할 때까지의 시간을 의미
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
헬스체크(Health Check)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
감시대상이 정상인 상태에 있는지 여부를 검사하는 것
<br>
<br>
</td>
</tr>
</table>

<br>

---

<br>

# 참고 자료

+ 서버/인프라를 지탱하는 기술 (책)

+ [ARP Flux 문제](https://www.joinc.co.kr/w/Site/System_management/GNS3Layer2)

+ [LVS 설정](https://www.oss.kr/storage/app/public/oss/13/66/021_[LVS]%20Solution%20Guide%20V0.4_20181204.pdf)

+ [keepalived 메뉴얼](https://keepalived.readthedocs.io/en/latest/)

+ [VRRP](https://limvo.tistory.com/13)

+ [리버스 프록시 장점(1)](https://narup.tistory.com/238)

+ [리버스 프록시 장점(2)](https://forteleaf.tistory.com/entry/Reverse-Proxy-%EC%99%80-Load-Balencer)

+ [프록시 종류와 장점](https://velog.io/@dev_leewoooo/Forward-Proxy-Reverse-Proxy%EC%97%90-%EA%B4%80%ED%95%98%EC%97%AC)

+ [SSL Passthrough](https://m.blog.naver.com/wideeyed/222070860143)

+ [SSL Termination](https://www.haproxy.com/blog/haproxy-ssl-termination/)

+ [HAProxy 메뉴얼](https://docs.haproxy.org/2.7/configuration.html)

+ [웹 캐시 종류](https://hudi.blog/http-cache/)

+ 캐시 서버 관련

    - [x] https://ko.linux-console.net/?p=672#gsc.tab=0

    - [x] https://luran.me/359

    - [x] https://brownbears.tistory.com/43
    
    - [x] https://haeunyah.tistory.com/80

    - [x] https://inpa.tistory.com/entry/DB-%F0%9F%93%9A-NoSQL-%EA%B0%9C%EB%85%90-%EC%A0%95%EB%A6%AC

    - [x] https://www.flexera.com/blog/cloud/2012/10/architecting-scalable-applications-in-the-cloud-the-caching-tier/

+ Apache MPM

    - [x] https://httpd.apache.org/docs/2.4/ko/mod/

    - [x] https://sailing-blog.com/6035

    - [x] https://jang8584.tistory.com/254

    - [x] https://junghyungil.tistory.com/118

    - [x] https://kchanguk.tistory.com/185

    - [x] https://meryer.tistory.com/m/1

    - [x] https://blogger.pe.kr/921

+ Apache Module mod_rewrite

    - [x] https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html

    - [x] https://snoop-study.tistory.com/10

    - [x] https://zetawiki.com/wiki/%EC%95%84%ED%8C%8C%EC%B9%98_mod_rewrite_RewriteRule_%ED%94%8C%EB%9E%98%EA%B7%B8

    - [x] https://co-no.tistory.com/63

    - [x] https://blog.munilive.com/posts/how-to-use-htaccess-rewrite-rule.html

+ Bonding 설정

    - [x] https://blog.naver.com/onevibe12/221717035000
    
    - [x] https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-using_channel_bonding
    
    - [x] https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_networking/configuring-network-bonding_configuring-and-managing-networking

+ STP 

    - [x] https://peemangit.tistory.com/25

    - [x] https://aws-hyoh.tistory.com/entry/Spanning-Tree-%EC%89%BD%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B02