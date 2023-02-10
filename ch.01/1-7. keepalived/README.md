## 목차

1-7-1. [keepalived](#1-7-1-keepalived)

1-7-2. [IPVS 구성 (NAT)](#1-6-2-ipvs-구성-nat)

---

# 1-7-1. keepalived

### ⓐ Keepalived

> 로드밸런싱과 고가용성(HA)을 제공하는 프레임워크

> Heartbeat 체크를 수행하다가 Master 노드에 장애가 발생하면 Slave 노드로 Failover(장애극복)하는 역할을 함

> 기본적으로는 L4 수준의 로드밸런싱을 지원하며, HAProxy와 함께 조합해서 사용하면 L7 로드밸런싱도 가능함

> VRRP을 사용한 고가용성(HA) 제공함

### ⓑ VRRP (Virtual Router Redundancy Protocol)

> 다중화(or 이중화) 구성을 하는 이유는 크게 두 가지로 다음과 같다.

>    > Load Balancing(부하분산) 

>    >  > 트래픽을 분산시켜 네트워크의 성능향상을 목표로 함

    + 고가용성(HA) 

      - 서비스를 중단 없이 지속적으로 제공하는 성질

      - Failover(장애극복) : 하나의 장비에 장애가 발생했을 때 다른 장비로 전환되어 서비스 단절을 최소화함

> "VRRP는 주로 Failover를 목적으로 Master/Backup 장비간의 전환을 위해 사용"된다. Keepalived의 HA는 자신의 IP 주소와는 별개로 VIP를 설정해두고 문제가 생겼을때 이 VIP를 다른곳으로 인계하여 같은 IP주소를 통해서 서비스가 지속되도록 해주는것이 핵심인데 이 부분은 Keepalived가 VIP 할당 및 해제를 자동으로 해주기 때문에 별도의 설정을 하지 않아도 문제가 없다.

> VRRP는 게이트웨이 이중화 프로토콜(FHRP : First Hop Redundancy Protocol) 중 하나로 게이트웨이 장애 복구를 위한 프로토콜로 다른 프로토콜로는 HSRP, GLBP 등이 있다.

    + 게이트웨이 이중화가 필요한 이유

      - End Device는 라우팅 기능이 없어 동적 라우팅 프로토콜을 설정할 수 없으므로, 게이트웨이 장애 시 이를 인지할 수 없으며 외부 네트워크와 통신할 수 없게 된다 (단일 장애 지점)

    + VRRP 동작 방식

      - 이중화 그룹을 대표하는 가상의 인터페이스를 생성하고 게이트웨이 IP(VIP)를 할당

      - 이중화 그룹에서 Master/Backup 장비를 결정하고 Master 장비는 라우팅 기능을 제공함과 동시에 VRRP Advertisement 패킷을 반복적으로 전송함으로써 Backup 장비에게 알림

      - 이때 Backup 장비는 해당 패킷을 받는 중에는 MAster 장비가 살아있다고 판단하여 Standby 상태를 유지