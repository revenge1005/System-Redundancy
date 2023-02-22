## 목차

3-1. [DNS 서버의 다중화](#3-1-dns-서버의-다중화)

3-2. [스토리지 서버의 다중화 (미러링 구성)](#3-2-스토리지-서버의-다중화-미러링-구성)

<br>

---

<br>

# 3-1. DNS 서버의 다중화

<br>

## ⓐ 주소변환 라이브러리를 이용한 다중화와 문제점 
> DNS를 다중화하려면 『/etc/resolv.conf』에 여러 DNS 서버를 지정하는 방법이 가장 손쉽지만 **『질의가 타임아웃된 경우 다음 네임서버에 질의해본다』라는 동작으로 인해 문제가 발생**한다. <br><br>
자세한 내용은 참조 - https://linux.die.net/man/5/resolv.conf <br><br>
**최초의 지정된 DNS 서버가 다운되면 타임아웃(디폴트 5초)을 대기한 후에 다음 서버로 질의를 한다 이 대기시간은 서버팜에서 심각한 성능저하**를 야기하는 요인이 된다.

<br>

#### 【 성능저하의 위험성 - 메일서버의 예 】

<br>

+ 메일서버가 메일을 송신할 때에는 DNS 질의를 2회 수행한다. <br><br>
    - [x] 목적지 주소의 도메인 파트에 대해 MX레코드 질의 <br><br>
    - [x] MX의 결과로부터 A레코드를 질의해서 송신 서버의 IP주소를 얻음 

<br>

+ 1시간에 1000통의 메일을 전송해야 하는 서버가 있을 경우, 최소 3초에 1통씩 메일을 전송할 수 있어야 한다. <br><br>

    - [x] /etc/resolv.conf에 지정된 DNS 서버 중 하나가 정지하면 DNS 질의 1회에 5초의 타임아웃이 발생 <br><br>

    - [x] 그렇게 되면 1통의 송신하는데 10초, 1시간에 처리할 수 있는 메일의 수는 대략 360통 되어 성능 저하 발생

<br>

#### 【 영향력이 큰 DNS 장애 】

<br>

+ 위 예시의 경우, 메일서버의 전송 성능이 현저히 저하되고 있음에도 전송 자체는 완료되므로 시스템은 정지되지 않는다.

<br>

+ DNS서버 장애는 영향을 미치는 범위가 크지만 장애가 발생한 장소를 찾아내려면 시간이 오래 걸리는 경우가 많다는 데 주의할 필요가 있다.

<br>

## ⓑ 서버팜에서의 DNS 다중화

<br>

#### 【 VRRP를 이용한 구성 】

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/42735894/220076539-ff8cc4a9-e13a-4025-a952-b8efce336877.PNG" width="500" height="381"/>
<p>

<br>

+ **이 상태에서는 Active서버의 keepalived가 정지하면 장애극복하지만, DNS서비스가 정지하더라도 장애극복하지 않는다**.<br><br>

+ 그러므로 헬스체크 스크립트를 실행해야하며, 아래 스크립트는 5초마다 dig명령을 사용해서 자기 자신에게 DNS질의를 해서 dig명령이 비정상 종료하면 keepalived를 정지시킨다.

    ```
    #!/bin/sh
    while true do
        /usr/bin/dig +time=001 +tries=3 @127.0.0.1 localhost.localnet
        if [ "0" -ne "$?" ]; then
            /etch/inint.d/keepalived stop
            exit
        fi
        sleep 5
    done
    ``` 

<br>

#### 【 DNS서버의 부하분산 】

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/42735894/220081544-23a3d9e6-0762-43b8-8407-c8bd13fa5c7c.PNG" width="800" height="443"/>
<p>

<br>

+ 로드밸런서는 리눅스에서 IPVS, keepalived를 사용해서 구축하고, 이 구성의 keepalived.conf 설정음 다음과 같다.
    ```
    virtual_server_group DNS {
        192.168.0.200 53
    }
    virtual_server group DNS {
        delay_loop 5
        lvs_sched rr
        lvs_method DR
        protocol UDP
        real_server 192.168.0.201 53 {
            weight 1
            MISC_CHECK {
                misc_path "/usr/bin/dig +time=001 +tries=3 @192.168.0.201 localhost.localnet"
                misc_timeout 5
            }
        }
        real_server 192.168.0.202 53 {
            weight 1
            MISC_CHECK {
                misc_path "/usr/bin/dig +time=001 +tries=3 @192.168.0.202 localhost.localnet"
                misc_timeout 5
            }
        }
    }
    ```

    - [x] keepalived에서는 DNS 헬스체크 기능이 지원되지 않으므로 MISC_CHECK를 사용해서 dig 명령을 실행해서 헬스체크하도록 한다.

<br>

---

<br>

# 3-2. 스토리지 서버의 다중화 (미러링 구성)

<br>

## ⓐ DRBD(Distributed Replicated Block Device)
> HA 클러스터를 구성하기 위한 블록 디바이스로, **네트워크를 통해서 두 대의 컴퓨터의 블록 디바이스 간에 데이터를 미러링**한다. 즉, 『네트워크를 이용한 RAID 1』이라고 생각하면 된다.

<br>

#### 【 DRBD의 구성 】

<br>

<p align="center">
<img src="https://user-images.githubusercontent.com/42735894/220126701-64cf3d7c-3d75-417b-9888-8256de5dc046.PNG" width="700" height="337"/>
<p>

<br>

+ DRBD는 커널모듈(디바이스 드라이버), 유저랜드(userland)툴로 구성되어 있다. 

    - [x] OS에서는 메모리 공간을 커널 공간과 사용자 공간으로 구분된다.

        - 커널 공간은 커널과 커널 확장 기능 그리고 장치 드라이버를 실행하기 위한 예비 공간

        - 유저 공간은 사용자 모드 응용 프로그램들이 동작하는 메모리 영역
    
    - [x] 유저랜드는 유저 공간에서 실행되는 프로그램 및 라이브러리를 가리킵니다.

<br>

+ 파일 단위로 데이터를 전송이 아닌 블록 디바이스에 대해 변경사항을 실시간으로 전송한다.

<br>

+ 파일 생성/변경할 때 DRBD나 백업 서버의 존재를 의식할 필요는 없다.

<br>

+ DRBD의 미러링은 Active/Backup 구성이으로 Active 측의 블록 디바이스에 대해서는 읽고 쓰기가 가능하지만, Backup 측의 블록 디바이스에는 접근할 수 없다.

<br>

## ⓑ DRBD의 설정과 실행
> 설정 파일은 /etc/drbd.d/ 에 넣으면 되며, /etc/drbd.conf 에서 /etc/drbd.d/ 하위의 설정파일을 불러 오는 방법을 쓴다.

<br>

#### 【 /etc/drbd.conf (Active/Backup) 】
```
# You can find an example in  /usr/share/doc/drbd.../drbd.conf.example
include "drbd.d/global_common.conf";
include "drbd.d/*.res";
```

<br>

#### 【 /etc/drbd.d/{name}.res (Active/Backup) 】
```
resource {name} {
    protocol { A | B | C };

    on {primary-hostname} {
        device      /dev/drbd0;
        disk        /dev/sdb1;
        address     192.168.0.201:7789;
        meta-disk   internal;
    }
    
    on {secondary-hostname} {
        device      /dev/drbd0;
        disk        /dev/sdb1;
        address     192.168.0.202:7789;
        meta-disk   internal;
    }
}
```

+ **resource :** 리소스를 정의하는 블록

<br>

+ **protocol :** 데이터 전송 프로토콜 지정
    
    - [x] **A :** 로컬 디스크에 쓰기가 끝나고 TCP 버퍼에 데이터를 송신한 시점에서 쓰기작업 완료 (성능을 중시하는 비동기 전송)
        
    - [x] **B :** 로컬 디스크에 쓰기가 끝나고 원격 호스트로 데이터가 도달한 시점에서 쓰기작업 완료 (A와 C의 중간)
        
    - [x] **C :** 원격 호스트의 디스크에도 쓰기가 완료된 시점에서 쓰기작업 완료 (신뢰성을 중시하는 동기화 전송)

<br>    

+ **on :** 호스트마다 리소스를 정의하는 블록

<br>    

+ **device :** DRBD의 논리 블록 디바이스를 지정

<br>    

+ **disk :** 미러링하고자 하는 물리 디바이스를 지정

<br>    

+ **address :** 데이터를 동기화하기 위해 수신 대기할 IP주소와 포트번호를 지정

<br>    

+ **meta-disk :** 메타 데이터를 저장할 디바이스를 지정
    
    - [v] **internal :** disk에서 지정한 블록 디바이스 중 일부를 메타 데이터용으로 확보

<br>

#### 【 DRBD 실행 (Active/Backup) 】
```
service drbd start
```

<br>

#### 【 DRBD Primary 만들기 (Active) 】
> DRBD 가 처음실행되면 secondary/secondary로 동작하며, 이를 primary/secondary로 만들어 mirroring이 진행될 수 있게 해야한다.

<br>

```
drbdadm primary {resource_name}
```

<br>

#### 【 DRBD 모니터링 】
```
$ cat /proc/drbd
version: 8.4.3 (api:1/proto:86-101)
srcversion: 88927CDF07AEA4F7F2580B2 
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:0 nr:43 dw:43 dr:1456 al:0 bm:2 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0


$ drbdadm role {resource_name}
Primary/Secondary


$ service drbd status
drbd driver loaded OK; device status:
version: 8.4.3 (api:1/proto:86-101)
srcversion: 88927CDF07AEA4F7F2580B2 
m:res  cs         ro                 ds                 p  mounted   fstype
0:r0   Connected  Primary/Secondary  UpToDate/UpToDate  C  /tmp/mnt  ext3 
```

<br>

#### 【 secondary를 primary 전환 】
```
# 1. primary의 문제 확인 (Unknow)

drbd-2$ cat /proc/drbd
version: 8.4.3 (api:1/proto:86-101)
srcversion: 88927CDF07AEA4F7F2580B2 
 0: cs:WFConnection ro:Secondary/Unknown ds:UpToDate/DUnknown C r-----
    ns:0 nr:43 dw:43 dr:1456 al:0 bm:2 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0

drbd-2$ service drbd status
drbd driver loaded OK; device status:
version: 8.4.3 (api:1/proto:86-101)
srcversion: 88927CDF07AEA4F7F2580B2 
m:res  cs            ro                 ds                 p  mounted  fstype
0:r0   WFConnection  Secondary/Unknown  UpToDate/DUnknown  C


# 2. secondary를 primary 전환

drbd-2$ drbdadm primary {resource_name}
```

<br>

## ⓒ DRBD 장애극복
> **DRBD는 마스터 서버에 문제가 발생하더라도 백업 서버가 자동적으로 마스터 서버가 되지는 않는다.** 따라서 keepalived를 이용해서 장애극복할 수 있도록 한다.

<br>

#### 【 DRBD의 장애극복 】

<br>

+ 장애극복하기 위해서는 마스터 서버의 DRBD를 secondary 상태로 하고 블록 디바이스가 마운트되어 있으면 실패하므로 NFS 서버를 정지해서 언마운트해둔다.

    ```
    cat <<EOF > /usr/local/sbin/drbd-backup
    /etc/init.d/nfs-kernel-server stop
    umount /mnt/drbd0
    drbdadm secondary all
    EOF
    ```

<br>

+ 백업 서버를 마스터로 할 경우 DRBD를 primary 상태로 하고 블록 디바이스를 마운트해서 NFS서버를 실행한다.

    ```
    cat <<EOF > /usr/local/sbin/drbd-master
    drbdadm primary all
    mount /dev/drbd0 /mnt/drbd0 
    /etc/init.d/nfs-kernel-server stop
    EOF
    ```

<br>

#### 【 DRBD - keepalived 설정 】
> VIP로 192.168.0.200으로, NFS 클라이언트는 192.168.0.200:/mnt/drbd0/을 NFS 마운트하고 있으며 서버가 장애극복해도 NFS 클라이언트느 다시 마운트할 필요가 없다.

![88787](https://user-images.githubusercontent.com/42735894/220269449-062f9bb6-873d-47c2-967c-68c1ea9cdacf.PNG)

```
vrrp_instance DRBD {
    state BACKUP
    interface eth0
    garp_master_delay 5
    virtual_router_id 200
    priority 100
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtaul_ipaddress {
        192.168.0.200/24 dev eth0
    }
    notify_master "/usr/local/sbin/drbd-master"
    notify_backup "/usr/local/sbin/drbd-backup"
    notify_fault "/usr/local/sbin/drbd-backup"
}
```
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
nopreempt
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
VRRP의 Preemptive mode를 무효화 설정, Preemptive 모드는 기존 마스터보다도 높은 우선순위를 갖는 노드가 기동하면 장애극복이 일어나는데 Preemptive 모드를 무효화하면 이미 마스터 노드가 가동 중일 경우 자신의 우선순위가 높더라도 장애극복을 하지 않는다. (즉, 불필요한 장애극복을 피하기 위해 설정함)
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
notify_master
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
VRRP가 마스터 상태가 됐을 때 실행하고자 하는 명령을 지정
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
notify_backup
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
VRRP가 백업 상태가 됐을 때 실행하고자 하는 명령을 지정
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
notify_fault
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
네트워크 인터페이스가 링크 다운되었을 때 실행하고자 하는 명령을 지정
<br>
<br>
</td>
</tr>
</table>

<br>

#### 【 keepalived가 종료할 때 】
> **keepalived가 종료할 때에는 notify_master/notify_backup의 스크립트는 실행되지 않는데** 이로 인해 마스터 서버의 keepalived가 정지하면 **마스터 서버의 DRBD가 primary 상태인 채로 장애극복하게 되므로, 백업 서버의 drbd-master는 에러가 발생한다.** <br><br>
이 문제를 해결하기 위해 daemontools 등의 툴을 이용하는데 supervise는 서비스를 관리하는 도구로, 서비스를 감시하여 서비스를 시작하고 만약 죽으면 재시작 시킨다. <br><br> 
Process supervision 관련 도구는 해당 내용 참고 - https://en.wikipedia.org/wiki/Process_supervision#Implementations

<br>

## ⓓ NFS서버를 장애극복할 때 주의점
> DRBD에서 미러링할 디바이스를 NFS로 공유하기 위해서는 마스터 서버에서 NFS 서버를 실행해야 한다.<br><br>
**장애극복으로 새롭게 마스터가 된 NFS서버는 아무 클라이언트에서도 마운트되어 있지 않다 그러나 NFS 클라이언트는 서버가 전환되었음을 알지 못하므로, 이미 마운트되었다고 보고 파일에 접근한다.**<br><br>
그 결과, **NFS서버에서는 "마운트되지 않은 클라이언트로부터 파일 접근 요청이 왔다"고 판단해서 접근을 거부하게 되는데 이를 해결하려면 다음과 같은 방법을 사용한다.**

<br>

1. **/var/lib/nfs/를 동기화한다**

    - [x] NFS 서버의 접속 정보는 /var/lib/nfs/에 저장되는데, DRBD로 이 볼륨을 미러링함으로써 장애극복을 해도 접속정보를 인계할 수 있다.

    - [x] 다만, 배포판에 따라서는 NFS 서버의 실행 스크립트 내에 exportfs 명령으로 접속정보를 초기화하는 경우가 있으며 이런 경우 접속 정보를 초기화하지 않는 실행 스크립트를 별도로 작성해서 장애극복할 때는 해당 스크립트로 NFS서버를 실행하도록 하는 방법이 필요하다.

<br>

2. **nfsd 파일시스템을 이용**

    - [x] nfsd 파일시스템은 NFS서버를 다중화하기 위해 만들어진 리눅스 고유의 기능이다.

    - [x] 『mount -t nfsd nfsd /proc/fs/nfsd』를 한 상태로 실행된 NFS서버는 /var/lib/nfs/ 디렉토리를 이용하지 않게 된다.

    - [x] 게다가 생면부지의 NFS클라이언트로부터 접속 요청이 있을 경우라도 이미 마운트되어 있는 듯이 처리할 수 있다.
    
<br>

---

<br>

# 3-3. L1, L2 네트워크의 다중화

<br>

## ⓐ 장애발생 포인트

1. **LAN 케이블**

2. **NIC**

3. **네트워크 스위치의 포트**

4. **네트워크 스위치**

+ **1~3 :** **서버와 스위치 간 접속에 장애가 발생할 수 있다 이를 이것을 『링크 장애』**, 

+ **1,3 :** **스위치끼리 연결하는 스위치 간 접속에 고장은 발생할 수 있다 이를 『스위치 간 접속 장애』**

+ **4 :** **네트워크 스위치의 고장은 "스위치 장애**

<br>

## ⓑ 링크 다중화와 Bonding 드라이브

<br>

#### 【 링크 다중화 】
> 링크 장애를 회피하기 위해서는 서버에 NIC를 여러 개 준비하고 LAN 케이블도 같은 수만큼 연결한다.<br><br>
**그러나 여러 NIC를 이용해 통신하기 위해서는 각각의 NIC에 대해 각기 다른 IP주소를 할당해야하고, 네트워크에 접속된 각 머신은 통신할 때마다 통신 상대의 어떤 NIC가 사용되는지를 진단해서 그 결과에 따라 송신할 주소를 전환해야 한다.**

<br>

#### 【 Bonding 】
> 여러 물리적인 네트워크 인터페이스를 하나의 논리적인 네트워크 인터페이스로 결합하여 네트워크 인터페이스를 만드는 데 사용되는 기술

<br>

+ 논리 NIC를 통한 통신은 Bonding 드라이버의 설정에 따라 하위의 물리 NIC에 할당된다 또한 하위의 물리 NIC에 장애여부를 체크해서 장애가 있으면 해당 물리 NIC는 사용하지 않도록 한다.

+ Bonding 드라이버가 여러 개 존재하는 물리 NIC 중에서 통신에 사용할 것을 선택하는 방법은 다음과 같다.

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
balance-rr 
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
송신할 패킷마다 사용할 물리 NIC를 전환한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
active-backup
</td>
<td width="80%">
<!-- REMOVE THE BACKSLASHES -->
<br>
첫 번째 물리 NIC가 사용가능한 동안에는 그 NIC만 사용한다. 해당 NIC가 고장 나면 다음 NIC를 사용한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
balance-xor
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
송신처와 목적지 MAC주소를 XOR해서 사용할 물리 NIC를 결정한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
broadcast
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
송신 패킷을 복사해서 모든 물리 NIC에 대해 동일한 패킷이 전송한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
802.3ad
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
IEEE 802.3ad 프로토콜을 사용해서 스위치와의 사이에서 동적으로 aggregation을 작성한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
balance-tlb
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
물리 NIC 중에 가장 부하가 낮은 물리 NIC를 선택해서 송신하단 수신은 특정 물리 NIC를 사용해 수신한다.
<br>
<br>
</td>
</tr>
<tr>
<td>
<!-- REMOVE THE BACKSLASHES -->
balancer-alb
</td>
<td>
<!-- REMOVE THE BACKSLASHES -->
<br>
송수신 모두 부하가 낮은 물리 NIC를 사용한다.
<br>
<br>
</td>
</tr>
</table>