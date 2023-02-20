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
<img src="https://user-images.githubusercontent.com/42735894/220081544-23a3d9e6-0762-43b8-8407-c8bd13fa5c7c.PNG" width="700" height="388"/>
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