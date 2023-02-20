## 목차

3-1. [DNS 서버의 다중화](#dns-서버의-다중화)

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