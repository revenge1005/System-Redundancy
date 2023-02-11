## 목차

1-7-1. [keepalived](#1-7-1-keepalived)

1-7-2. [keepalived 설정](#1-7-2-keepalived-설정)

## Guide

+ https://keepalived.readthedocs.io/en/latest/

+ https://fossies.org/linux/keepalived/doc/man/man5/keepalived.conf.5

<br>

---

<br>

# 1-7-1. keepalived

<br>

## ⓐ Keepalived

+ 로드밸런싱과 고가용성(HA)을 제공하는 프레임워크

+ Heartbeat 체크를 수행하다가 Master 노드에 장애가 발생하면 Slave 노드로 Failover(장애극복)하는 역할을 함

+ 기본적으로는 L4 수준의 로드밸런싱을 지원하며, HAProxy와 함께 조합해서 사용하면 L7 로드밸런싱도 가능함

+ VRRP을 사용한 고가용성(HA) 제공함

<br>

## ⓑ VRRP (Virtual Router Redundancy Protocol)

> 다중화(or 이중화) 구성을 하는 이유는 크게 두 가지로 다음과 같다.

- [x] **Load Balancing(부하분산) :** 트래픽을 분산시켜 네트워크의 성능향상을 목표로 함 <br><br>
- [x] **고가용성(HA) :** 서비스를 중단 없이 지속적으로 제공하는 성질 <br><br>

> VRRP는 주로 Failover를 목적으로 Master/Backup 장비간의 전환을 위해 사용된다. Keepalived의 HA는 자신의 IP 주소와는 별개로 VIP를 설정해두고 문제가 생겼을때 이 VIP를 다른곳으로 인계하여 같은 IP주소를 통해서 서비스가 지속되도록 해주는것이 핵심인데 이 부분은 Keepalived가 VIP 할당 및 해제를 자동으로 해주기 때문에 별도의 설정을 하지 않아도 문제가 없다.

<br>

> VRRP는 게이트웨이 이중화 프로토콜(FHRP : First Hop Redundancy Protocol) 중 하나로 게이트웨이 장애 복구를 위한 프로토콜로 다른 프로토콜로는 HSRP, GLBP 등이 있다.
+ **게이트웨이 이중화가 필요한 이유** <br>
    - [x] End Device는 라우팅 기능이 없어 동적 라우팅 프로토콜을 설정할 수 없으므로, 게이트웨이 장애 시 이를 인지할 수 없으며 외부 네트워크와 통신할 수 없게 된다. (단일 장애 지점)

<br>

+ **VRRP 동작 방식** <br>
    - [x] 이중화 그룹을 대표하는 가상의 인터페이스를 생성하고 게이트웨이 IP(VIP)를 할당 <br><br>
    - [x] 이중화 그룹에서 Master/Backup 장비를 결정하고 Master 장비는 라우팅 기능을 제공함과 동시에 VRRP Advertisement 패킷을 반복적으로 전송함으로써 Backup 장비에게 알림 (이때 Backup 장비는 해당 패킷을 받는 중에는 MAster 장비가 살아있다고 판단하여 Standby 상태를 유지함) <br><br>
    - [x] Master 장비에 장애가 발생해 Backup 장비에게 VRRP Advertisement 패킷을 수신하지 못하면, Backup 장비들은 서로 VRRP Advertisement 패킷과 GARP 패킷을 교환하여 우선 순위에 따라 새로운 Master 장비를 선정 <br><br>
       - GARP는 자신의 IP에게 ARP 요청을 보내는 것으로 동일 서브넷 상에 존재하는 장비의 ARP Table을 갱신할 수 있다. <br><br>
       - 여기서 Backup과 연결된 스위치나 라우터로 GARP 패킷을 통해 ARP Table을 갱신해 Backup 장비가 VIP를 소유하게 되었음을 알릴 수 있다.

<br>

> + **Master/Backup 선출 기준** 
>   - VIP와 RIP가 같은 장비
>   - VRRP Priority(우선순위) 값이 큰 장비
>   - RIP의 주소가 큰 장비

<br>

### ⓒ Keepalived 에서 제공하는 헬스체크

> + TCP_CHECK : 비동기식으로 Time-Out TCP 요청을 통해 장애를 검출하는 방식 <br><br>
> + HTTP_GET : HTTP GET 요청을 보내서 서비스의 정상 동작을 확인 <br><br>
> + SSL_GET : HTTPS GET 요청을 보내서 서비스의 정상 동작을 확인 <br><br>
> + MISC_CHECK : 특정 기능을 확인하는 스크립트를 실행하여 결과가 0인지 1인를 가지고 검출하는 방식

<br>

---

<br>

# 1-7-2. keepalived 설정

<br>

### ⓐ Keepalived 설치

> ```
> yum -y install keepalived
> ```

<br>

### ⓑ Keepalived 설정 (lv1, lv2 노드 모두 설정)

> ```
> cat <<EOF >> /etc/keepalived/keepalived.conf
> vrrp_instance VI_1 {
>	    state MASTER
>	    interface eth0
>	    garp_master_delay 5
>	    virtual_router_id 200
>	    priority 101	# lv2는 100으로 변경할 것
>	    advert_int 1
>	    authentication {
>	        auth_type PASS
>	        auth_pass 1111
>	    }
>	    virtual_ipaddress {
>	        10.0.0.100/24	dev  eth0
>	        192.168.219.100/24	dev  eth1
>	    }
> }
> EOF
> ```

<br>

### ⓒ VRRP 인스턴스 분리

> + 위 설정에서는 VRRP 패킷은 eth0으로만 전송되므로 eth1의 LAN 케이블을 빼더라도 이상 작동을 검출할 수 없다. <br><br>
> + 여러 인터페이스에서 장애를 검출하고자 할 경우에는 인스턴스마다 VRRP 인스턴스를 정의할 필요가 있다. <br><br>
> + VRRP 인스턴스는 virtual_router_id 파라미터에에 짜라 나뉘어지므로, 인스턴스마다 고유한 값을 지정하기 바란다. 
> ```
>  cat <<EOF >> /etc/keepalived/keepalived.conf
>  vrrp_instance VE {
>	    state MASTER
>	    interface eth0
>	    garp_master_delay 5
>	    virtual_router_id 200	# VRRP 인스턴스마다 고유한 값
>	    priority 101		# lv2는 100으로 변경할 것
>	    advert_int 1
>	    authentication {
>	        auth_type PASS
>	        auth_pass 1111
>	    }
>	    virtual_ipaddress {
>	        10.0.0.100/24	dev  eth0
>	    }
>  }
>  vrrp_instance VI {
>	    state MASTER
>	    interface eth0
>	    garp_master_delay 5
>	    virtual_router_id 200	# VRRP 인스턴스마다 고유한 값
>	    priority 101		# lv2는 100으로 변경할 것
>	    advert_int 1
>	    authentication {
>	        auth_type PASS
>	        auth_pass 1111
>	    }
>	    virtual_ipaddress {
>	        192.168.219.100/24	dev  eth1
>	    }
>  }
>  EOF
> ```