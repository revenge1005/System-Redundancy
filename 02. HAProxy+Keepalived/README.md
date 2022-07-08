## A) LVS-01, LVS-02 Node - HAProxy Installation and setup 

### A-1) HAProxy Install
```
apt update

apt-get install haproxy -y
```

### A-2) HAProxy Setup
```
# net.ipv4.ip_nonlocal_bind 옵션은 프로그램이 시스템 상의 장치에 없는 주소로 Binding 할 수 있도록 할 수 있게하는 커널 파라미터
# HAProxy 및 Keepalived의 로드밸런싱은 동시에 로컬이 아닌 IP 주소에 바인딩할 수 있어야 한다
# 즉, 네트워크 인터페이스에 등록되지 않은 주소로 바인딩할 수 있도록 하는 커널 값이다. 
# (네트워크 인터피에스가 지정된 정적 IP가 아닌 동적 IP를 바인딩할 수 있다.)
# 해당 옵션이 비활성화 되어 있어도 서비스가 시작하면서 인터페이스에 특정 IP를 바인딩할 수 있으나 장애극복(Failover)시 문제 발생


echo 'net.ipv4.ip_nonlocal_bind=1' >> /etc/sysctl.conf
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p


cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.org

cat <<EOF >> /etc/haproxy/haproxy.cfg

frontend LOADBALANCER-01
        bind                    192.168.219.100:80
        mode                    http
        default_backend         WEBSERVERS-01

backend WEBSERVERS-01
        balance         roundrobin
        option          httpchk GET /l7check.html HTTP/1.0
        option          log-health-checks
        option          forwardfor
        option          httpclose
        cookie          SERVERID        rewrite
        cookie          JSESSIONID      prefix
        balance         roundrobin
        server          web-01  192.168.219.21:80 cookie web-01 check inter 1000 rise 2 fall 5
        server          web-02  192.168.219.22:80 cookie web-02 check inter 1000 rise 2 fall 5

listen  SERVER-STATS    # Define a listen section called "SERVER-STATS"
        bind            192.168.219.11:9000             # Listen on port 9000
        mode            http
        stats           enable                          # Enable stats page
        stats           hide-version                    # Hide HAProxy Version
        stats           realm   Haproxy\ Statistics     # Title Text for popup window
        stats           uri     /haprxoy_stats          # Stats URI
        stats           auth    admin:admin             # authentication credentials
EOF
```

### A-3) Service start/enable
```
haproxy -f /etc/haproxy/haproxy.cfg -c

systemctl enable haproxy
systemctl restart haproxy
```

----

## B) LVS-01, LVS-02 Node - Keepalived Installation and setup 

### B-1) Keepalived Install
```
apt update 

apt -y install keepalived
```

### B-2) Keepalived Setup - LVS-01(Master) Setup
```
cat <<EOF >> /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state MASTER
    interface ens32
    virtual_router_id 51
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.219.12
    unicast_peer {
        192.168.219.11
    }
    virtual_ipaddress {
        192.168.219.100/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### B-3) Keepalived Setup - LVS-02(Backup) Setup
```
cat <<EOF >> /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    unicast_src_ip 192.168.219.11
    unicast_peer {
        192.168.219.12
    }
    virtual_ipaddress {
        192.168.219.100/24
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

### B-4) Service start/enable
```
systemctl enable keepalived
systemctl restart keepalived
```

### B-5) Check the result

1) Failover

![01](https://user-images.githubusercontent.com/42735894/177974884-b534a482-0a28-4c67-836e-0999aaa10146.PNG)
![02](https://user-images.githubusercontent.com/42735894/177974905-f0b7627c-7379-494f-9a86-053222393a69.PNG)
![03](https://user-images.githubusercontent.com/42735894/177974916-2d4b64a1-3a5a-4eb5-b79d-0e7e99eeff78.PNG)

2) Failback - 다시 LVS-01 서버 작동 시키면 기존의 Master가 복귀한 것을 확인할 수 있다

![01](https://user-images.githubusercontent.com/42735894/177974884-b534a482-0a28-4c67-836e-0999aaa10146.PNG)