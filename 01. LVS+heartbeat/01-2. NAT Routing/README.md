
## A) LVS-01, LVS-02 Node - HeartBeat Installation and setup 

### A-1) HeartBeat Install
```
apt update

apt-get install heartbeat -y
```

### A-2) HeartBeat Setup
```
cp /usr/share/doc/heartbeat/authkeys /etc/ha.d/
cp /usr/share/doc/heartbeat/ha.cf.gz /etc/ha.d/
cp /usr/share/doc/heartbeat/haresources.gz /etc/ha.d/
gunzip -d /etc/ha.d/ha.cf.gz
gunzip -d /etc/ha.d/haresources.gz
chmod 600 /etc/ha.d/authkeys
```

### A-3) authkeys - 노드간의 인증 방법 설정
```
sed -i "s/#auth 1/auth 1/g" /etc/ha.d/authkeys
sed -i "s/#1 crc/1 crc/g" /etc/ha.d/authkeys
```

### A-4) ha.cf - heartbeat 구성에 필요한 기본 설정 파일
```
cat <<EOF >> /etc/ha.d/ha.cf

# debugfile 로그 생성 파일
debugfile /var/log/ha-debug

# 로그 생성 파일
logfile /var/log/ha-log

# 두 노드간 heartbeat 주고받는 주기(초단위)
keepalive 2

# 노드가 죽었다고 판단하는 시간
deadtime 30

# 자동 복구 On (한 노드가 서비스 하다가 죽었을 때 새로운 노드로 자원이 이동)
# 기본적으로 설정되어 있음
# auto_failback on ()

initdead 120

# UDP heartbeat 패킷을 보낼 포트
udpport 694

# 노드간 체크할 사용할 인터페이스
bcast ens32

# 노드 등록
node lvs-01
node lvs-02
EOF

cat <<EOF >> /etc/hosts

192.168.219.11 lvs-01
192.168.219.12 lvs-02
EOF
```

### A-5) haresource - 노드간에 공유할 자원과 스크립트 설정 파일
```
cat <<EOF >> /etc/ha.d/haresources

# Write both lvs-01 and lvs-02 as lvs-01.
lvs-01 IPaddr::192.168.219.100/24/ens32:0/192.168.219.255
lvs-01 IPaddr::192.168.56.100/24/ens34:0/192.168.56.255
EOF
```

### A-6) Service Start/Enable
```
systemctl enable heartbeat
systemctl start heartbeat
```

<br>

----

## B) LVS-01, LVS-02 Node - LVS Installation and setup 

### B-1) ipvsadm Install
```
apt update 

apt -y install ipvsadm
```

### B-2) IP forwarding Setup
```
sed -i "s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g" /etc/sysctl.conf
sysctl -p
```

### B-3) ipvsadm Setup
```
ipvsadm -C
ipvsadm -A -t 192.168.219.100:80 -s rr
ipvsadm -a -t 192.168.219.100:80 -r 192.168.56.21:80 -m
ipvsadm -a -t 192.168.219.100:80 -r 192.168.56.22:80 -m

ipvsadm-save
```

### B-4) ipvsadm Save Settings Permanently
```
cat <<EOF > /etc/systemd/system/ipvs-config.service
[Unit]
Description=Configure IPVS
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ipvsadm -C
ExecStart=/sbin/ipvsadm -A -t 192.168.219.100:80 -s rr
ExecStart=/sbin/ipvsadm -a -t 192.168.219.100:80 -r 192.168.56.21:80 -m
ExecStart=/sbin/ipvsadm -a -t 192.168.219.100:80 -r 192.168.56.22:80 -m
ExecStart=/sbin/ipvsadm-save
RemainAfterExit=false
StandardOutput=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable ipvs-config
```

### B-5) ipvsadm Setup Check
```
ipvsadm -Ln --rate
```

<br>

----

## C) WEB-01, WEB-02 - Apache Installation and setup

### C-1) Apache Install
```
apt update 

apt -y install apache2
```

### C-2) Gateway IP address change
```
cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:
            addresses: [192.168.56.21/24]
            gateway4: 192.168.56.100
            nameservers:
                    addresses: [168.126.63.1, 8.8.8.8]
  version: 2
```

<br>

----

## D) Check the results
![result](https://user-images.githubusercontent.com/42735894/177139535-806222ca-67f2-4c79-818e-365342e32711.PNG)