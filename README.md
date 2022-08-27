# VPN
1. Между двумя виртуалками поднять vpn в режимах:
- tun
- tap
Описать в чём разница, замерить скорость между виртуальными машинами в туннелях, сделать вывод об отличающихся показателях скорости.
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку.
3 (*). Самостоятельно изучить, поднять ocserv и подключиться с хоста к виртуалке.

## Среда выполнения
```
root@yarkozloff:/otus/vpn# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
  
root@yarkozloff:/otus/vpn# vboxmanage --version
6.1.26_Ubuntur145957

root@yarkozloff:/otus/vpn# vagrant --version
Vagrant 2.3.0
```
+ Локально загруженный бокс centos/7
## 1. VPN между двумя виртуальными машинами
Пишем Vagrantfile, который поднимает 2 машины и пишем роли для развертывания VPN. В playbook будет осуществляться следующие операции:

### Для server машины:
- Установка epel-release
- Установка пакетов tcpdump, policycoreutils-python, openvpn, iperf3
- Отключение selinux
- Генерация openvpn ключа
- Копирование openvp конфига
- Старт сервиса openvp
- Экспорт сгенерированного openvpn ключа
Серверный openvpn/server.conf:
```
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
### Для client машины:
- Установка epel-release
- Установка пакетов tcpdump, policycoreutils-python, openvpn, iperf3
- Отключение selinux
- Копирование openvpn ключа
- Копирование openvp конфига для client
- Старт сервиса openvp
Клиентский openvpn/server.conf:
```
dev tap
remote 192.168.10.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.10.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
### Замеряем скорость в туннеле
## TAP
- на openvpn сервере запускаем iperf3 в режиме сервера: iperf3 -s &
```
[root@server ~]# iperf3 -s &
[1] 5570
[root@server ~]# -----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.10.10.2, port 44482
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 44484
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec  2.10 MBytes  17.6 Mbits/sec
[  5]   1.00-2.00   sec  2.64 MBytes  22.2 Mbits/sec
[  5]   2.00-3.00   sec  2.69 MBytes  22.6 Mbits/sec
[  5]   3.00-4.00   sec  2.65 MBytes  22.2 Mbits/sec
[  5]   4.00-5.01   sec  2.52 MBytes  21.0 Mbits/sec
[  5]   5.01-6.00   sec  2.49 MBytes  21.0 Mbits/sec
[  5]   6.00-7.00   sec  2.52 MBytes  21.1 Mbits/sec
[  5]   7.00-8.00   sec  2.52 MBytes  21.1 Mbits/sec
[  5]   8.00-9.00   sec  2.28 MBytes  19.1 Mbits/sec
[  5]   9.00-10.00  sec  2.27 MBytes  19.0 Mbits/sec
[  5]  10.00-11.01  sec  2.53 MBytes  21.1 Mbits/sec
```
- на openvpn клиенте запускаем iperf3 в режиме клиента и замеряем скорость в туннеле: 
```
[vagrant@client ~]$ iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  4] local 10.10.10.2 port 44484 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.00   sec  14.5 MBytes  24.3 Mbits/sec    0    587 KBytes
[  4]   5.00-10.00  sec  13.7 MBytes  23.0 Mbits/sec    0   1.09 MBytes
[  4]  10.00-15.00  sec  12.4 MBytes  20.8 Mbits/sec   38   1.07 MBytes
[  4]  15.00-20.00  sec  13.6 MBytes  22.9 Mbits/sec   11    939 KBytes
[  4]  20.00-25.00  sec  14.8 MBytes  24.9 Mbits/sec    0   1.07 MBytes
[  4]  25.00-30.00  sec  12.3 MBytes  20.7 Mbits/sec    0   1.09 MBytes
```
### TUN
Пробуем тоже самое в режиме tun заменив в конфиге tap на tun:
server:
```
[root@server ~]# iperf3 -s &
[2] 5631
[root@server ~]# iperf3: error - unable to start listener for connections: Address already in use
iperf3: exiting
Accepted connection from 10.10.10.2, port 44486
[  5] local 10.10.10.1 port 5201 connected to 10.10.10.2 port 44488
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec  2.87 MBytes  24.1 Mbits/sec
[  5]   1.00-2.00   sec  2.68 MBytes  22.5 Mbits/sec
[  5]   2.00-3.00   sec  2.83 MBytes  23.7 Mbits/sec
[  5]   3.00-4.00   sec  2.80 MBytes  23.4 Mbits/sec
[  5]   4.00-5.00   sec  2.98 MBytes  25.0 Mbits/sec
[  5]   5.00-6.00   sec  2.72 MBytes  22.8 Mbits/sec
[  5]   6.00-7.01   sec  2.99 MBytes  25.0 Mbits/sec
[  5]   7.01-8.00   sec  3.01 MBytes  25.4 Mbits/sec
[  5]   8.00-9.00   sec  3.13 MBytes  26.2 Mbits/sec
[  5]   9.00-10.00  sec  2.97 MBytes  24.9 Mbits/sec
[  5]  10.00-11.00  sec  3.08 MBytes  25.8 Mbits/sec
```
client:
```
[root@client ~]# iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  4] local 10.10.10.2 port 44488 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.00   sec  15.9 MBytes  26.6 Mbits/sec    0    654 KBytes
[  4]   5.00-10.00  sec  14.6 MBytes  24.5 Mbits/sec   28    458 KBytes
[  4]  10.00-15.01  sec  15.5 MBytes  25.9 Mbits/sec    0    552 KBytes
[  4]  15.01-20.00  sec  15.4 MBytes  25.9 Mbits/sec    0    595 KBytes
^C[  4]  20.00-20.95  sec  2.29 MBytes  20.3 Mbits/sec    0    626 KBytes
```
### Отличия tun и tap нитерфесов:
- tap - уровень L2, tun - L3
- tun - не умеет в мультикаст => не позволяет использовать OSPF (link-state протоколы динамической маршрутизации). tap - умеет.
## 2. RAS на базе OpenVPN с клиентскими сертификатами
Подготавливаем машину client и server и устанавливаем пакеты openvpn, easy-rsa:
```
Vagrant.configure(2) do |config|
 config.vm.box = "centos7"
 config.vm.define "rasvpn" do |rasvpn|
 rasvpn.vm.hostname = "rasvpn.loc"
 rasvpn.vm.network "private_network", ip: "192.168.10.10"
 config.vbguest.auto_update = false
 config.vm.provision "ansible" do |ansible|
        ansible.playbook = "play_serverras.yaml"
      end
 end
 config.vm.define "client" do |client|
 client.vm.hostname = "client.loc"
 client.vm.network "private_network", ip: "192.168.10.20"
 config.vbguest.auto_update = false
 config.vm.provision "ansible" do |ansible|
        ansible.playbook = "play_clientras.yaml"
      end
 end
end
```
Далее действия выполняются вручную.

Подключаемся к машине rasvpn, накоторой будет работать серверная часть openvpn, переходим в директорию /etc/openvpn/ и инициализируем pki:
```
[root@localhost ~]# cd /etc/openvpn/
[root@localhost openvpn]# /usr/share/easy-rsa/3.0.8/easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /etc/openvpn/pki
```
Генерируем необходимые ключи и сертификаты для сервера и клиента
```
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass  
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass  
echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server  
/usr/share/easy-rsa/3.0.8/easyrsa gen-dh  
openvpn --genkey --secret ta.key  
echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass  
echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client  
```
Зададим параметр iroute для клиента
```
echo 'iroute 192.168.10.20 255.255.255.0' > /etc/openvpn/client/client
```
Создаем конфигурационный файл /etc/openvpn/server.conf для сервера
```
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
route 192.168.10.0 255.255.255.0
push "route 192.168.10.0 255.255.255.0"
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
#comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
Конфигурационный файл /etc/openvpn/client.conf для клиента:
```
dev tun
proto udp
remote 192.168.10.10 1207
client
resolv-retry infinite
#ca ./ca.crt
#cert ./client.crt
#key ./client.key
route 192.168.10.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3
```
Теперь когда всё готово, кладем необходимые сертификаты/ключи на client машину в каталог /etc/openvpn или прописываем ключи как указано выше. Запускаемся:
```
openvpn --config client.conf
```
При успешном подключении проверяем пинг в внутреннему IP адресу сервера в туннеле.
```
[root@client openvpn]# ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=63 time=3.31 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=63 time=15.2 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=63 time=38.9 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=63 time=4.09 ms

--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 3.313/15.397/38.946/14.391 ms
```
Также проверяем командой ip r (netstat -rn) на клиент машине, что сеть туннеля импортирована в таблицу маршрутизации.
``[root@client openvpn]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.20 metric 101`

```
