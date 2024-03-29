## 목차

1-1. [다중화](#1-1-다중화)

1-2. [웹 서버의 다중화 - DNS 라운드로빈](#1-2-웹-서버의-다중화---dns-라운드로빈)

1-3. [웹 서버 다중화 - IPVS를 이용한 로드밸런서](#1-3-웹-서버-다중화---ipvs를-이용한-로드밸런서)

1-4. [L4 스위치의 NAT 구성과 DSR 구성](#1-4-l4-스위치의-nat-구성과-dsr-구성)

1-5. [동일 서브넷인 서버를 부하분산할 경우 주의사항](#1-5-동일-서브넷인-서버를-부하분산할-경우-주의사항)

1-6. [(실습) ipvsadm](https://github.com/revenge1005/System-Redundancy/tree/master/ch.01/1-6.%20ipvsadm)

1-7. [(실습) keepalived](https://github.com/revenge1005/System-Redundancy/tree/master/ch.01/1-7.%20keepalived)

1-8. [(실습) LVS+heartbeat 구성](https://github.com/revenge1005/System-Redundancy/tree/master/ch.01/1-8.%20LVS%2Bheartbeat)

1-9. [(실습) HAProxy+Keepalived](https://github.com/revenge1005/System-Redundancy/tree/master/ch.01/1-9.%20HAProxy%2BKeepalived)

<br>

---

<br>

# 1-1. 다중화

> 다중화란, 장애가 발생해도 예비 운용장비로 시스템의 기능을 계속할 수 있도록 하는 것을 말한다.

<br>

## ⓐ 라우터 장애 시의 경우

#### 【 Cold Standby 】

> 예비 운용장비는 사용하지 않고 있다가 현재 운용장비에 장애가 발생하면 예비 운용장비를 연결하는 운용 형태 

- [x] 주의해야할 점은 현재 장비와 예비 장비의 설정은 동일하게 해야한다는 점이다. 

- [x] 라우터와 같은 네트워크 장비는 운영중에 빈번히 설정을 변경할 일도 없고, 저장할 데이터가 별로 없으므로 Cold Standby가 적절하다

<br>

## ⓑ 웹 서버 장애 시의 경우

#### 【 Hot Standby 】
> 두 대의 서버를 항상 가동시켜 두고 늘 같은 상태로 유지해두는 운용 형태

<br>

## ⓒ 장애극복(Failover)
> 현재 운용장비에 장애가 발생했을 때 자동으로 예비 운용장비로 처리를 인계하는 것

<br>

#### 【 가상 IP 주소(VIP) 】
> 현재 운용장비인 웹 서버에는 자신의 IP주소와는 별개로 VIP를 할당해 두고 서비스는 VIP로 제공함 

<br>

#### 【 IP 주소 인계 】
> 현재 운용장비에 장애가 발생했을 때에는 예비 운용장비가 VIP를 인계한다.

<br>

## ⓓ 장애 검출 (Health Check)
> 정상적으로 장애극복하기 위해서는 현재 운용장비에서 장애가 발생하고 있음을 검출하는 방법

<br>

#### 【 ICMP 감시 (Layer3) 】

> ICMP의 Echo 요청을 보내서 응답이 돌아오는지를 체크 

- [x] 가장 간단한 방법이지만, 웹서비스가 다운된 경우는 감지할 수 없다.

<br>

#### 【 포트 감시 (Layer4) 】

> TCP로 접속을 시험해서 접속할 수 있는지 여부를 체크 

- [x] 웹 서비스가 다운된 것은 감지할 수 있지만, 과부화 상태로 응답할 수 없다거나 에러를 반환하는 것은 감지할 수 없다.

<br>

#### 【 서비스 감시 (Layer7) 】

> 실제로 HTTP 요청 등을 보내서 정상적인 응답이 들어오는지 체크 

- [x] 대부분의 이상을 감지할 수 있지만, 경우에 따라서는 서버에 부하를 유발할 수도 있다.

<br>

#### 【 웹 서버의 헬스체크 】

> "서버 장애에 의한 서비스 중지"는 정상적으로 검출하기 위해서는 "서비스 감시"를 이용한다.

- [x] 서비스 감시를 이용하는 이유는 서버의 전원이 켜져 있어서 ICMP 응답이 돌아오더라도 서비스가 정상적으로 동작하고 있다고 할 수 없기 때문이다.

<br>

#### 【 라우터의 헬스체크 】

> "라우터 장애에 의한 서비스 정지"를 검출하기 위해서는 ICMP 감시를 이용한다. 

- [x] 라우터에 대한 감시를 하는 것이 아니라 확실히 패킷을 전송할 수 있는가이므로 웹 서버에게 통신할 수 있는 상태인지를 확인할 수 있으면 된다

<br>

---

<br>

# 1-2. 웹 서버의 다중화 -  DNS 라운드로빈 
<br>

## ⓐ DNS를 이용해서 하나의 서비스에 여러 대의 서버를 분산시키는 방법

1.  유저 A와 B는 각각 DNS 서버에 "www.test.com"의 IP 주소를 질의하면 DNS 서버는 "x.x.x.1", "x.x.x.2"라는 서로 다른 IP 주소를 반환한다. <br><br>
2. 그 결과 A는 "x.x.x.1"로, B는 "x.x.x.2"로 접속한다. <br><br>
3. DNS 서버는 동일한 이름으로 여러 레코드를 등록시키면 질의할 때마다 다른 결과를 반환한다. <br><br>
4. 이 동작을 이용함으로써 여러 대의 서버에 처리를 분산시킬 수가 있다.

<br>

## ⓑ DNS 라운드로빈 문제

#### 【 서버의 수만큼 글로벌 주소가 필요 】 
> 수많은 서버로 부하분산하기 위해서는 IP주소를 많이 얻을 수 있는 서비스를 이용할 필요가 있다.

<br>

#### 【 균등하게 분산되는 것은 아님 】 
>   **⒜ 모바일 사이트에서 문제가 되는 경우**
- [x] 스마트폰으로부터의 접속은 캐리어 게이트웨이라고 하는 프록시 서버를 경우한다. <br><br>
- [x] 프록시 서버에서는 이름변환 결과가 일정시간 동안 캐싱되므로 같은 프록시 서버를 경유하는 접속은 항상 같은 서버로 전달된다. <br><br>
- [x] 따라서 균등하게 접속이 분산되지 않고 특정 서버만 처리가 집중할 가능성이 있다. <br><br>
>   **⒝ PC 웹 브라우저**
- [x] PC 웹브라우저도 DNS 질의 결과를 캐싱하기 때문에 균등학 부하분산되지 않는다. <br><br>
- [x] DNS 레코드의 TTL을 짧게 설정함으로써 어느 정도 개선할 수는 있지만, TTL에 따라 캐시를 해제하는 것은 아니므로 주의할 필요가 있다. 

<br>

#### 【 서버가 다운돼도 감지하지 못함 】   
> DNS 서버는 웹 서버의 부하나 접속수 등의 상황에 따라 질의결과를 제어할 수 없다. 즉, 서버가 다운되더라도 이를 검출하지 못하고 계속 부하분산을 하며 이로 이내 다운된 서버로 분산된 유저는 에러 페이지를 접하게 된다.

<br>

---

<br>

# 1-3. 웹 서버 다중화 - IPVS를 이용한 로드밸런서
<br>

## ⓐ DNS 라운드로빈과 로드밸런서의 차이
+ 로드밸런서는 하나의 IP주소에 대해 요청을 복수의 서버로 분산할 수 있다. <br><br>
+ DNS 라운드로빈에서는 웹 서버마다 다른 글러벌 주소를 할당해야 했지만, 로드밸런서를 이용하면 글로벌 주소를 절약할 수 있다.

<br>

#### 【 로드밸런서의 동작 】 
+ 로드밸런서는 서비스용 글러벌 주소를 가진 가상서버로서 동작한다. <br><br>
+ 그리하여 클라이언트로부터 전송된 요청을 실제 웹 서버로 중계함으로써 마치 자신이 웹 서버인 것처럼 작동한다.

<br>

#### 【 로드밸런서의 기능 】 
+ 로드밸런서는 여러 대의 리얼서버 중에 한 대를 선택해서 처리를 중계한다. <br><br>
+ 이 때, 헬스체크가 실패하는 서버는 선택되지 않고 반드시 헬스체크가 성공한 서버를 선택한다. <br><br>
+ 따라서, 특정 서버 한대가 정지해 있더라도 정상적으로 가동하고 있는 서버가 있는 한 서비스가 정지하지 않는다.

<br>

## ⓑ LVS(Linux Virtual Server)
> 고가용성 서버를 구축하기 위해 리눅스 머신을 로드밸런스 하도록 해주는 운영 시스템

<br>

## ⓒ IPVS (리눅스로 로드밸런서 구성)
> Netfilter Framework 기반으로 구현된 리눅스 커널 레벨에서 동작하는 L4 로드밸런싱 도구
- [x] **⒜ 동작 원리** <br><br>

    1. 패킷을 효과적으로 추적하고 라우트하기 위해 IPVS는 리눅스 커널에 IPVS 테이블을 생성함 (해당 테이블은 해쉬 테이블)

    2. IPVS 테이블은 ipvsadm 명령을 통해서 지속적으로 업데이트가 가능함

    3. 해당 테이블 목록에 따라 특정 End Point로 설정된 스케줄링 알고리즘을 사용해서 부하분산을 수행함

<br>

- [x] **⒜ IPVS 구축 방식** : Direct Routing, NAT, Tunneling  

<br>

## ⓓ 로드밸런서는 크게 L4스위치와 L7스위치 두 종류가 있다.
> "L4 스위치"는 전송계층까지의 정보를 분석하므로 IP주소나 포트번호에 따라 분산대상 서버를 지정할 수가 있다. (IPVS) <br><br>
"L7 스위치"는 애플리케이션 계층까지의 정보를 분석하므로 클라이언트부터 요청된 URL에 따라 분산대상 서버를 지정할 수가 있다.

<br>

## ⓔ L4와 L7 스위치의 동작 차이
> "L4 스위치"에서는 클라이언트가 통신하는 곳은 리얼서버이지만, "L7 스위치"에서는 하나의 접속에 대해 클라이언트<->로드밸런서와 로드밸런서<->리얼서버의 두 TCP 세션이 전개된다. <br><br>
"L4 스위치"와 "L7 스위치"의 특징을 단적으로 정리하면 유연한 설정을 하고자 하면 "L7 스위치", 성능을 추구하면 "L4 스위치"를 사용한다.

<br>

## ⓕ 스케줄링 알고리즘
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
rr(round-robin)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
리얼 서버를 처음부터 차례로 선택하여 모든 서버로 균등하게 처리가 분산된다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
wrr(weighted round-robin)
</td>
<td width="70%">
<!-- REMOVE THE BACKSLASHES -->
<br>
rr과 같지만 가중치를 가미해서 분산비율을 변경한다. (가중치가 큰 쪽을 빈번히 선택함)
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
lc(least-connection)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
접속수가 가장 적은 서버를 선택 (어떤 것을 사용하면 좋을지 모를 경우에 사용해도 좋다)
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
wlc(weighted least-connection)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
+ lc와 같지만 가중치를 가미해서 분산비율을 변경한다. <br>
<br>
+ 『(접속수+1)/가중치』가 최소가 되는 서버를 선택 <br>
<br>
+ 고성능 서버는 가중치를 크게 하는 것이 좋음
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
sed(shortest expected delay)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
+ 가장 응답속도가 빠른 서버를 선택 <br>
<br>
+ 서버와의 응답시간을 계측하는 것이 아닌 ESTABLISHED 상태인 접속수가 적은 서버를 선택 <br>
<br>
+ wlc와 동일한 동작하지만 wlc에서는 ESTABLISHED 이외의 상태인 접속수를 더하는 점이 다름 <br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
nq(naver queue)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
sed와 동일한 알고리즘이지만 active 접속수가 0인 서버를 최우선으로 선택
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
sh(source hashing)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
소스 IP주소로부터 해시값을 계산해서 분산대상 리얼서버를 선택
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
dh(destination hashing)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
목적지 IP주소로부터 해시값을 계산해서 분산대상 리얼서버를 선택
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
lblc<br>(locality-based least-connection)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
+ 접속수가 가중치로 지정한 값을 넘기 전까지는 동일한 서버를 선택 <br>
<br>
+ 모든 서버의 접속수가 가중치로 지정한 값을 넘을 경우, 마지막에 선택된 서버가 계속 선택
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
 lblcr(lblc replication)
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
lblc와 동일하며 접속수가 적은 서버로 선택됨
<br>
<br>
</td>
</tr>
</table>

<br>

---

<br>

# 1-4. L4 스위치의 NAT 구성과 DSR 구성
<br>

## ⓐ NAT(Network Address Translation) 구성
> L4 스위치는 클라이언트로부터 도착한 패킷의 수신 주소를 변경해서 리얼 서버로 전송한다. 이를 위해 응답 패킷을 받아서 IP주소를 원래대로 되돌릴 필요가 있다.

<br>

## ⓑ DSR(Direct Server Return)
> L4 스위치는 클라이언트로부터 수신한 패킷을 그대로 리얼 서버로 라우팅한다. 이 경우, 응답패킷에 대해 IP 주소를 되돌릴 필요가 없으므로 리얼서버는 L4 스위치를 경유하지 않고 응답할 수 있다. 

> DSR 구성에서는 가상서버를 향한 패킷(글로벌 주소로의 패킷)이  리얼서버에 그대로 도달하므로 리얼서버가 글로벌 주소를 처리할 수 있다. 즉, NAT 구성으로 동작하고 있는 시스템에 대해 로드밸런서의 설정만 DSR로 변경한다고 해도 부하분산을 할 수 없다, 설정방법은 다음과 같다. 
- [x] **리얼서버의 루프백 인터페이스에 가상서버의 IP주소를 할당하는 방법 <br><br>**
- [x] **netfilter을 이용해서 가상서버를 향한 패킷을 리얼서버 자신을 향한 것처럼 목적지 주소를 변경하는 방법 (DNAT)**

<br>

---

<br>

# 1-5. 동일 서브넷인 서버를 부하분산할 경우 주의사항
<br>

## ⓐ 동일 서브넷의 웹 서버에서 메일서버에게 대량의 메일을 송신하기 위한 부하분산하는 경우
+ 동일한 서브넷인 서버에 대해 부하분산을 하고자 할 경우는 NAT 구성을 사용할 수 없다. <br><br>
+ NAT구성에서는 로드밸런서가 목적지 IP주소를 변경하기에 메일서버가 받아들이는 패킷은 수신측 주소가 "메일서버", 송신측 주소가 "웹서버" 된다 따라서, 메일 서버가 반환하는 응답패킷의 송신측 "메일서버"는 동일한 서브넷인 IP 주소이므로 로드밸런서로 가지 않고 웹서버로 직접 송신된다. <br><br>
+ 그 결과 NAT에 의해 변경된 IP 주소를 원래대로 되돌릴 수가 없으므로 정상적으로 통신할 수 없게 된다. <br><br>
+ DSR 구성으로 하면 해결되며 DSR의 경우 로드밸런스는 IP 주소를 변경하지 않으므로 메일서버가 직접 웹 서버에 응답을 반환해도 문제없다