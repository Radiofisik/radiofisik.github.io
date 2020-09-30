# OpenVpn

Настройка сервера

```bash
push "route 192.168.1.0 255.255.255.0"
server 172.16.0.0 255.255.255.0
route 172.16.0.0 255.255.255.0
route 192.168.2.0 255.255.255.0
route 192.168.0.0 255.255.255.0

dev tun0
proto udp
port 1194

keepalive 15 60
daemon
verb 3

client-to-client
duplicate-cn  

auth md5
#cipher AES-128-CBC
cipher bf-cbc
comp-lzo

mtu-test

dh /tmp/openvpn/dh.pem
ca /tmp/openvpn/ca.crt
cert /tmp/openvpn/cert.pem
key /tmp/openvpn/key.pem

# Only use crl-verify if you are using the revoke list - otherwise leave it commented out
# crl-verify /tmp/openvpn/ca.crl

# management parameter allows DD-WRT\s OpenVPN Status web page to access the server\s management port
# port must be 5001 for scripts embedded in firmware to work
management localhost 5001

client-config-dir /jffs/openvpn/ccd
ifconfig-pool-persist /jffs/openvpn/ccd/ipp.txt

#fixes performance issues
sndbuf 393216
rcvbuf 393216
push "sndbuf 393216"
push "rcvbuf 393216"

```

Проверка статуса подключения

```bash
root@RFNet:/# telnet localhost 5001
>INFO:OpenVPN Management Interface Version 1 -- type 'help' for more info
status
OpenVPN CLIENT LIST
Updated,Sat Sep  7 00:06:48 2019
Common Name,Real Address,Bytes Received,Bytes Sent,Connected Since
work,78.36.19.40:51849,82154,81924,Fri Sep  6 23:50:35 2019
notebook,83.29.16.6:26672,79807,83121,Fri Sep  6 23:50:32 2019
ROUTING TABLE
Virtual Address,Common Name,Real Address,Last Ref
172.16.0.10,notebook,83.29.16.6:26672,Sat Sep  7 00:06:28 2019
172.16.0.6,work,78.36.19.40:51849,Fri Sep  6 23:50:38 2019
GLOBAL STATS
Max bcast/mcast queue length,0
END

```

фиксирование адреса клиента

```bash
echo "ifconfig-push 172.16.0.6 172.16.0.5">/jffs/openvpn/ccd/work
echo "iroute 192.168.0.0 255.255.255.0">>/jffs/openvpn/ccd/work

echo "ifconfig-push 172.16.0.10 172.16.0.9">/jffs/openvpn/ccd/notebook
echo "work,172.16.0.6">/jffs/openvpn/ipp.txt
echo "notebook,172.16.0.10">>/jffs/openvpn/ipp.txt

#в конфиг добавим
client-config-dir /jffs/openvpn/ccd
ifconfig-pool-persist /jffs/openvpn/ipp.txt
```



Настройка клиента

```bash
client
dev tun
proto udp
remote www.radiofisik.ru 1194
resolv-retry infinite
nobind

persist-key
persist-tun

ca ca.crt
cert work.crt
key work.key

remote-cert-tls server

auth md5
#cipher AES-128-CBC
cipher bf-cbc
comp-lzo

mtu-test

tun-mtu 8192
mssfix 1300

verb 3
topology subnet
```

