## 목차

1-6-1. [IPVS 구성 (Direct Routing)](#1-6-1-ipvs-구성-direct-routing)

1-6-2. [IPVS 구성 (NAT)](#1-6-2-ipvs-구성-nat)

<br>

---

<br>

# 1-6-1. IPVS 구성 (Direct Routing)

### ⓐ DR 구성 개요

+ Real Server와 Dispatcher Node가 가상 IP 주소를 공유한다.

+ Dispatcher Node와 Real Server는 네트워크 인터페이스에 VIP가 설정되 있어야 한다.

+ 이 인터페이스를 이용해 Dispatcher Node는 요청 패킷을 받아들이고 스케줄링에 의해 선택된 리얼 서버로 직접 라우팅한다.

<br>

### ⓑ 동작 방식

1. 클라이언트는 VIP로 서비스 요청을 하면 Dispatcher 노드는 Real Server들 중 하나로 스케줄링 한다.

2. 선택된 서버로 직접 라우팅해 클라이언트의 여청을 Real Server로 전달한다.

3. Real Server는 요청 사항을 처리한 후, Dispatcher Node를 거치지 않고 클라이언트로 직접 응답을 한다.

4. 각 Real Server는 Dispatcher와 같은 가상 IP를 공유하고 있기 때문에 Dispatcher 노드를 거치지 않고 클라이언트로 직접 응답할 수 있다

<br>

## ⓒ Dispatcher Node 설정

> 클라이언트로부터 서비스 요청 패킷을 받아 미리 설정된 스케줄링 방식으로 리얼 서버에게 부하를 분산하는 역할을 수행

<br>

#### 【 VIP 및 DIP 】

> Dipatcher Node에서는 크게 DIP(Direct IP)와 VIP(Virtual IP)를 설정해야 한다. (DIP는 Dipatcher Node가 고유하게 가질 IP 주소, VIP는 외부 클라이언트에게 서비스를 제공하기 위해 사용될 IP 주소)
```
ifconfig eth0:1 <Virtual IP> netmask 255.255.255.0 up
```

<br>

#### 【 Packet Forwarding 활성화 】

> Dispatcher는 클라이언트로부터 전송된 요청 메시지를 Real Server로 전달해야 하므로, 반드시 Packet Forwarding 기능을 활성화 해야 한다.
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

<br>

#### 【 Virtual Service 등록 (ipvsadm 설정) 】
```
yum -y install ipvsadm
```
```
ipvsadm <옵션>
```
<table>
<tr>
<th align="center">
<img width="441" height="1">
<p> 
<small>
옵션
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
-A
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
가상 서비스의 추가를 위한 옵션, 임의의 가상 서비스는 IP 주소, Port 번호, 프로토콜
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
-t 
</td>
<td width="70%">
<!-- REMOVE THE BACKSLASHES -->
<br>
TCP 서비스를 의미하며, 가상 서비스를 규정하는 한 요소
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
-s
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
TCP 연결과 UDP Datagram을 Real Server에게 전달하기 위한 알고리즘 설정
<br>
<br>
</td>
</tr>
</table>

<br>

## ⓓ Real Server 설정

> 실제 Client에게 서비스를 제공하는 노드로서, Dispatcher Node로부터 전달된 요청을 처리한 뒤 바로 클라이언트에게 응답을 전송한다.

<br>

#### 【 ARP Flux 문제 】

> 동일한 서브넷에 있는 두 개의 인터페이스를 가진 호스트가 동일한 서브넷의 인터페이스에 대한 ARP 요청을 동일한 서브넷의 모든 NIC에서 응답할 때 발생하는 문제를 말한다. <br><br>
예를 들자면 호스트(A), eth0(A.A.A10) / 호스트(B), eth0(A.A.A.20), eth1(A.A.A.30)가 있고,  호스트(A)와 호스트(B) 사이에서 통신할 때 방화벽 등의 이유로 호스트(B)의 eth1 인터페이스를 거쳐야 한다고 가정할 때 호스트(A)에서 호스트(B)의 eth1 인터페이스로의 트래픽이 eth0 인터페이스에서 대신 처리된다.

<br>

#### 【 ARP Flux 해결 - ARP Hidden 방식 】

> **패킷 전달 방식에 있어 NAT를 제외한 다른 방식은 모두 ARP Flux 문제를 해결해야 한다.** 기본적으로 Dispatcher와 Real Server 모두 VIP를 가지고 있기 때문에, 클라이언트가 VIP의 MAC 주소를 묻는 ARP 요청을 전송했을 때 Dispatcher와 Real Server 모두 응답하게 되면 클라이언트는 이들 중 특정 서버의 MAC 주소를 해당 VIP에 해당하는 MAC 주소로 기억하고는 이 서버에게만 모든 서비스 요청을 전송한다. <br><br>
결국 부하분산을 더 이상 제공할 수 없으며, 임의의 클라이언트가 어떤 서버로부터 서비스를 제공받는지 관리할 수 없다. **이를 해결하기 위해 클라이언트가 ARP 요청을 전송했을 때 Dispatcher만이 이에 응답하고, Real Server는 모두 ARP 요청을 무시해야 한다.**

```
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

<br>

#### 【 VIP 및 RIP 설정 】

> Real Server는 고유의 IP 주소인 RIP(Real Server IP)를 갖고 있으며, Dispatcher는 RIP 주소를 기반으로 서비스 요청을 분산시킨다. **(DR 방식의 특성상 Client의 요청은 Real Server에서 클라이언트로 직접 전달되므로 Real Server는 VIP 주소도 함께 lo 장치로 갖고 있어야 한다.)**
```
ifconfig lo:0 <Virtual IP> netmask 255.255.255.255 up
```

<br>

#### 【 Routing Table 설정 】

> lo:0 장치에 VIP를 설정했으므로, 목적지가 VIP인 요청 메시지를 lo:0 장치로 전달해야 한다.

```
route add -host <Virtual IP> dev lo:0
```

<br>

## ⓔ Real Server를 Virtual Service에 등록 (on Dispatcher Node)

#### 【 Dispatcher를 Real Server로서 등록하기 】
> Dispatcher가 Real Server로서 Virtual Service를 제공하기 위해서는 Client로부터 요청 메시지를 Local host로 전송할 수 있도록 등록해야 함
```
ipvsadm -a -t <Virtual IP>:80 -r 127.0.0.1 -g
```
<table>
<tr>
<th align="center">
<img width="441" height="1">
<p> 
<small>
옵션
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
-r 
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
Virtual Service에 등록할 Real Server의 IP주소 또는 호스트명으로서, 경우에 따라 Port 번호를 추가할 수 있다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
-g 
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
Direct Routing
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
-i 
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
 Tunneling
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
-m
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
NAT
<br>
<br>
</td>
</tr>
</table>

<br>

#### 【 Real Server 등록하기 】
```
ipvsadm -a -t <Virtual IP>:80 -r <Virtual IP> -g
```

<br>

#### 【 설정 확인 】
```
ipvsadm -Ln
```

---

# 1-6-2. IPVS 구성 (NAT)

## ⓐ NAT 구성 개요

> NAT 방식은 패킷 내의 IP 주소를 변경해 부하분산을 수행하는 방법

<br>

## ⓑ 동작 방식

1. 클라이언트에게는 Dispatcher의 도메인 네임 또는 IP가 알려져 있고, 클라이언트가 이 알려진 도메인 네임이나 IP를 사용해 Dispatcher에게 서비스 요청 패킷을 전송한다. 

2. 또한 Dispatcher는 N개의 Real Server 중 하나를 정해진 스케줄링 방법에 의해 선택한 후 패킷 내의 목적지 주소를 해당 서버의 IP로 다시 작성한다. 

3. Real Server는 클라이언트의 요청을 처리한 후, Dispatcher에게 응답을 돌려주고 이때 Dispatcher Node는 실제 응답의 발신자 주소를 다시 자신의 IP로 변경한 후 클라이언트에 서비스를 제공한다.

<br>

## ⓒ Dispatcher Node 설정

#### 【 VIP 】
```
ifconfig eth0:1 <Virtual IP> netmask 255.255.255.0 up
```

<br>

#### 【 Packet Forwarding 활성화 】
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

<br>

#### 【 Virtual Service 등록 (ipvsadm 설정) 】
```
ipvsadm -A -t <Virtula IP>:80 -s rr
```

### ⓓ 【 Real Server를 Virtual Service에 등록 (on Dispatcher Node) 】

#### 【 Real Server 등록하기 】
```
ipvsadm -a -t <VIP>:80 -r <RIP> -g
```

<br>

#### 【 Real Server 등록하기 】
```
ipvsadm -Ln	
```