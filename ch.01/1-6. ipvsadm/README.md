# 1-6-1. IPVS 구성 (Direct Routing)
<br>

### ⓐ DR 구성 개요

```
+ Real Server와 Dispatcher Node가 가상 IP 주소를 공유한다.

+ Dispatcher Node와 Real Server는 네트워크 인터페이스에 VIP가 설정되 있어야 한다.

+ 이 인터페이스를 이용해 Dispatcher Node는 요청 패킷을 받아들이고 스케줄링에 의해 선택된 리얼 서버로 직접 라우팅한다.
```

### ⓑ 동작 방식

```
+ 클라이언트는 VIP로 서비스 요청을 하면 Dispatcher 노드는 Real Server들 중 하나로 스케줄링 한다.

+ 선택된 서버로 직접 라우팅해 클라이언트의 여청을 Real Server로 전달한다.

+ Real Server는 요청 사항을 처리한 후, Dispatcher Node를 거치지 않고 클라이언트로 직접 응답을 한다.

+ 각 Real Server는 Dispatcher 노드와 같은 가상 IP를 공유하고 있기 때문에 Dispatcher 노드를 거치지 않고 클라이언트로 직접 응답할 수 있다
```

### ⓒ Dispatcher Node 설정

```
클라이언트로부터 서비스 요청 패킷을 받아 미리 설정된 스케줄링 방식으로 리얼 서버에게 부하를 분산하는 역할을 수행
```

#### 【 VIP 및 DIP 】
```
+ Dipatcher Node에서는 크게 DIP(Direct IP)와 VIP(Virtual IP)를 설정해야 한다.

+ DIP는 Dipatcher Node가 고유하게 가질 IP 주소, VIP는 외부 클라이언트에게 서비스를 제공하기 위해 사용될 IP 주소
```
```
ifconfig eth0:1 <Virtual IP> netmask 255.255.255.0 up
```

#### 【 Packet Forwarding 활성화 】
```
Dispatcher는 클라이언트로부터 전송된 요청 메시지를 Real Server로 전달해야 하므로, 반드시 Packet Forwarding 기능을 활성화 해야 한다.
```
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

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

### ⓓ Real Server 설정

```
실제 클라이언트에게 서비스를 제공하는 노드로서, Dispatcher Node로부터 전달된 요청을 처리한 뒤 바로 클라이언트에게 응답을 전송한다.
```

#### 【 ARP Flux 문제 】
```
+ 동일한 서브넷에 있는 두 개의 NIC를 가진 호스트가 동일한 서브넷의 NIC에 대한 ARP 요청을 동일한 서브넷의 모든 NIC에서 응답할 때 발생함

+ 호스트(A), eth0(A.A.A10) / 호스트(B), eth0(A.A.A.20), eth1(A.A.A.30)가 있고, 

+ 호스트(A)와 호스트(B) 사이에서 통신할 때 방화벽 등의 이유로 호스트(B)의 eth1 인터페이스를 거쳐야 한다고 가정할 때

+ 호스트(A)에서 호스트(B)의 eth1 인터페이스로의 트래픽이 eth0 인터페이스에서 대신 처리된다.
```

#### 【 ARP Flux 해결 - ARP Hidden 방식 】
```
+ 패킷 전달 방식에 있어 NAT를 제외한 다른 방식은 모두 ARP Flux 문제를 해결해야 한다.

+ 기본적으로 Dispatcher와 Real Server 모두 VIP를 가지고 있기 때문에, 클라이언트가 VIP의 MAC 주소를 묻는 ARP 요청을 전송했을 때 

+ Dispatcher와 Real Server 모두 응답하게 되면 클라이언트는 이들 중 특정 서버의 MAC 주소를 해당 VIP에 해당하는 MAC 주소로 기억하고는 

+ 이 서버에게만 모든 서비스 요청을 전송한다.

+ 결국 부하분산을 더 이상 제공할 수 없으며, 임의의 클라이언트가 어떤 서버로부터 서비스를 제공받는지 관리할 수 없다.

+ 이를 해결하기 위해서는 클라이언트가 ARP 요청을 전송했을 때 Dispatcher만이 이에 응답하고, Real Server는 모두 ARP 요청을 무시해야 한다.
```
```
echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

#### 【 VIP 및 RIP 설정 】
```
+ Real Server는 고유의 IP 주소인 RIP(Real Server IP)를 갖고 있으며, Dispatcher는 RIP 주소를 기반으로 서비스 요청을 분산시킨다

+ DR 방식의 특성상 Client의 요청은 Real Server에서 클라이언트로 직접 전달되므로 Real Server는 VIP 주소도 함께 lo 장치로 갖고 있어야 한다
```
```
ifconfig lo:0 <Virtual IP> netmask 255.255.255.255 up
```

#### 【 Routing Table 설정 】
```
+ lo:0 장치에 VIP를 설정했으므로, 목적지가 VIP인 요청 메시지를 lo:0 장치로 전달해야 한다.
```
```
route add -host <Virtual IP> dev lo:0
```

### ⓔ Real Server를 Virtual Service에 등록 (on Dispatcher Node)

#### 【 Dispatcher를 Real Server로서 등록하기 】
```
Dispatcher가 Real Server로서 Virtual Service를 제공하기 위해서는 Client로부터 요청 메시지를 Local host로 전송할 수 있도록 등록해야 함
```
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