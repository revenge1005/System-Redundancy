
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
EOF
```

### A-6) Service Start/Enable
```
systemctl enable heartbeat
systemctl start heartbeat
```

## A) LVS-01, LVS-02 Node - LVS
