---
title: "IPSec между Cisco и Linux, при помощи systemd-networkd и VTI"
date: 2022-09-17T17:41:36+03:00
author: Vadim Aleshin
tags: ['networking', 'linux', 'cisco']
---

В [предыдущей]({{< ref "/posts/eve-ng-linux-iso.md" >}}) статье был создан образ роутера на Linux для EVE-NG на основе Debian 11. Теперь, используя данный образ, настроим популярный сценарий - IPSec между роутером Cisco и роутером с Linux.  

Существуют несколько методов организации IPSec туннеля между двумя точками:
1. Policy-based - трафик, который должен отправляться в туннель, определяется на основе политик в процессе конфигурации IPSec'а.
2. Route-based - трафик, который должен отправляться в туннель, определяется с помощью маршрута по адресу назначения и конфигурируется с помощью протоколов маршрутизации (либо статических маршрутов), обычно поверх VTI или GRE туннельных интерфейсов.

В этой статье будет использоваться Route-based IPSec VPN с VTI интерфейсами.  

## Топология и устройства

Соберем простую топологию, как на изображении ниже.  

![ipsec-topology](/img/ipsec-topology.png)

Адресация:
1. 1.1.1.0/24 - сеть провайдера для точки с роутером Cisco
2. 2.2.2.0/24 - сеть провайдера для точки с роутером Linux
3. 3.3.3.3/32 - loopback интерфейс, имитирующий удаленную сеть
4. 10.10.10.0/30 - сеть для туннельных интерфейсов  
5. 192.168.100.0/24 - локальная сеть за роутером Cisco  
6. 192.168.200.0/24 - локальная сеть за роутером Linux  

## Настройка роутера на Linux

Будет использоваться:
1. Для настройки сети - **systemd-networkd**
2. Для настройки IPSec - **strongswan**
3. Для настройки OSPF - **frr**

### Настройка сети с systemd-networkd

По умолчанию Debian 11 использует файлы из директории **/etc/network** для конфигурации сети.  
Т.к. мы будем использовать systemd-networkd, то первым делом переименуем стандартный конфигурационный файл и отключим соответствующий сервис:  

> Необходимо не забывать, что после этого пропадет удаленный доступ, если он был. Так что все действия необходимо делать, имея локальный доступ.

```
mv /etc/network/interfaces /etc/network/interfaces.save
systemctl disable networking.service
systemctl stop networking.service
```
Далее включим и запустим сервис:
```
systemctl enable systemd-networkd
systemctl start systemd-networkd
```
Конфигурационные файлы хранятся в **/etc/systemd/network/**, т.к. директория по умолчанию пустая, то нам необходимо создать файлы с расширениями: **.network** - для конфигурации сетевых параметров, **.netdev** - для создания виртуальных интерфейсов.  

#### Настройка WAN интерфейса

1. Создадим файл с любым названием и расширением .network (systemd проверяет файлы по алфавитному порядку, поэтому удобней использовать число в названии, для управления последовательностью запуска)
```
touch 10-wan.network
```
2. Зададим настройки подключения (т.к. в текущей лабораторной нет DNS-сервера, то строку с настройкой закомментирую) 
```
[Match]
Name=ens3

[Network]
Address=2.2.2.2/24
Gateway=2.2.2.254
#DNS=2.2.2.254
IPForward=ipv4
IPMasquerade=ipv4
Tunnel=vti0
```
> **ens3** - имя интерфейса, который смотрит в сторону провайдера (на топологии в EVE-NG он e0, но это неверно, нумерация, в моем случае, начинается с ens3), в другом образе имя может быть другим.

> **Tunnel=vti0** - имя туннельного интерфейса, который создадим следующим шагом

> В реальном мире настройка NAT будет производиться с помощью nftables или iptables, в этом примере будет использоваться IPMasquerade=ipv4 (хотя в этой лабораторной NAT не нужен вообще)

Перезагрузим сервис, после чего доступ в интернет (в нашем случае до loopback интерфейса на роутере ISP) должен появиться.  
```
networkctl reload
```
```
ping 3.3.3.3
PING 3.3.3.3 (3.3.3.3) 56(84) bytes of data.
64 bytes from 3.3.3.3: icmp_seq=1 ttl=254 time=1.27 ms
64 bytes from 3.3.3.3: icmp_seq=2 ttl=254 time=1.92 ms
64 bytes from 3.3.3.3: icmp_seq=3 ttl=254 time=1.58 ms
```

#### Настройка LAN интерфейса

Настройка для LAN интерфейса проста. Создадим файл с расширением .network и просто назначим IP адрес на интерфейс, смотрящий в сторону ПК (ens4).  
```
touch 15-lan.network
```
```
[Match]
Name=ens4

[Network]
Address=192.168.200.1/24
```
```
networkctl reload
```

#### Настройка VTI интерфейса

Для использования в Route-based VPN нам необходимо создать VTI интерфейс, для этого:
1. Создадим файл виртуального устройства, с расширением .netdev 
```
touch 25-vti0.netdev
```
Зададим следующую конфигурацию, в секции [NetDev] укажем название, тип интерфейса и MTU.  
В секции [Tunnel] - адрес локального интерфейса, удаленного роутера и ключ (данный ключ будет использоваться в настройках strongswan, для определения трафика, который пойдет через vti интерфейс).  
```
[NetDev]
Name=vti0
Kind=vti
MTUBytes=1400

[Tunnel]
Local=2.2.2.2
Remote=1.1.1.1
Key=69
```

> Имя vti0 используется выше в конфигурации wan интерфейса, эти имена должны совпадать

2. Создадим .network файл, отвечающий за конфигурацию vti интерфейса  
```
touch 25-vti0.network
```
Просто задаём IP адрес для интерфейса vti0.  
```
[Match]
Name=vti0

[Network]
Address=10.10.10.2/30
```

### Настройка IPSec

Для настройки необходим установленный пакет **strongswan**.  

1. Отключаем установку маршрутов при помощи IKE демона, т.к. маршрутизацию мы настроим далее, с помощью пакета frr.  
Для этого в  файле **/etc/strongswan.d/charon.conf** находим строку с параметром **install_routes**, раскомментируем ее и установим значение в no.  
```
install_routes = no
```
2. В файле **/etc/ipsec.conf** настроим параметры IPSec, значения left и leftsubnet используются для локальных адресов, right и rightsubnet - для удаленных.  
```
conn vti-to-cisco
        left=2.2.2.2
        leftsubnet=0.0.0.0/0
        right=1.1.1.1
        rightsubnet=0.0.0.0/0
        ike=aes256-sha2_256-modp1024
        esp=aes-sha2_256
        authby=secret
        auto=start
        keyexchange=ikev2
        lifetime=1440m
        mark=69
        type=transport
```
3. В файле **/etc/ipsec.secrets** укажем пароль для данного туннеля.  
```
2.2.2.2 : PSK 'IPSEC_KEY'
```
4. В файле **/etc/sysctl.conf** необходимо добавить строку для отключения поиска политик для vti интерфейса, т.к. данный интерфейс используется в route-based, а не policy-based VPN.  
Необходимо обратить внимание на имя интерфейса в данной строке (**vti0**), оно должно совпадать с ранее созданным именем vti.
```
net.ipv4.conf.vti0.disable_policy=1
```
5. Перезагрузим сервисы и на этом настройка на роутере с Linux заканчивается (маршрутизацию настроим позже).  
```
systemctl restart systemd-networkd.service
ipsec restart
```

## Настройка роутера Cisco

Вдаваться в детальные настройки не будем, просто приведу последовательность.  

1. Настройка WAN и LAN интерфейсов и маршрута по умолчанию
```
configure terminal
 interface Ethernet0/1
 ip address 1.1.1.1 255.255.255.0
 ip nat outside
exit
 interface Ethernet0/0
 ip address 192.168.100.1 255.255.255.0
 ip nat inside
 exit
```
```
ip route 0.0.0.0 0.0.0.0 1.1.1.254
```

2. Настройка NAT (в лабораторной он не нужен, просто для полноты картины)
```
ip access-list extended NAT
 permit ip 192.168.100.0 0.0.0.255 any
 exit
ip nat inside source list NAT interface Ethernet0/1 overload
```

3. Настройка IPSec
```
crypto ikev2 proposal LINUX_PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 2

crypto ikev2 policy LINUX_POLICY
 proposal LINUX_PROPOSAL

crypto ikev2 keyring LINUX_KEYRING
 peer LINUX
  address 2.2.2.2
  pre-shared-key IPSEC_KEY
 
crypto ikev2 profile LINUX_PROFILE
 match identity remote address 2.2.2.2 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local LINUX_KEYRING

crypto ipsec transform-set TSET esp-aes esp-sha256-hmac
 mode transport

crypto ipsec profile LINUX_IPSEC
 set transform-set TSET
 set ikev2-profile LINUX_PROFILE
```

4. Настройка VTI интерфейса
```
interface Tunnel0
 ip address 10.10.10.1 255.255.255.252
 ip mtu 1400
 tunnel source Ethernet0/1
 tunnel mode ipsec ipv4
 tunnel destination 2.2.2.2
 tunnel protection ipsec profile LINUX_IPSEC
```

## Проверка работы IPSec

На Linux роутере:
```
root@linuxrouter# ipsec status
Security Associations (1 up, 0 connecting):
vti-to-cisco[1]: ESTABLISHED 117 minutes ago, 2.2.2.2[2.2.2.2]...1.1.1.1[1.1.1.1]
vti-to-cisco{5}:  INSTALLED, TUNNEL, reqid 2, ESP SPIs: c87f9062_i 080e098c_o
vti-to-cisco{5}:   0.0.0.0/0 === 0.0.0.0/0
vti-to-cisco{6}:  INSTALLED, TRANSPORT, reqid 1, ESP SPIs: c3a18c45_i 6489ad73_o
vti-to-cisco{6}:   2.2.2.2/32 === 1.1.1.1/32
```
```
root@linuxrouter# ip tunnel show vti0
vti0: ip/ip remote 1.1.1.1 local 2.2.2.2 dev ens3 ttl inherit nopmtudisc key 69
```
```
root@linuxrouter# ping -c 2 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=255 time=0.930 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=255 time=0.865 ms
--- 10.10.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.865/0.897/0.930/0.032 ms
```

На роутере Cisco:
```
CiscoRouter#show crypto ikev2 sa remote 2.2.2.2
Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         1.1.1.1/4500          2.2.2.2/4500          none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:2, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/7232 sec
```
```
CiscoRouter#show ip int br tu0
Interface                  IP-Address      OK? Method Status                Protocol
Tunnel0                    10.10.10.1      YES NVRAM  up                    up
```
```
CiscoRouter#ping 10.10.10.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.10.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/7 ms
```

## Настройка OSPF

    На Linux роутере будет использоваться пакет frr, он должен быть установлен (в собранном для EVE-NG образе он уже присутствует).  

Первым делом необходимо включить демон ospf. Для этого в файле **/etc/frr/daemons** выставить значение **ospfd=yes**, после чего перезапустить frr.service.  
```
systemctl restart frr.service
```
Теперь перейдем в интерактивный shell пакета frr, который имеет Cisco-like синтаксис. Для этого наберем **vtysh**.  
Детальной настройки делать не будем, просто укажем сети, которые используются в OSPF процессе: 

```
configure terminal
router ospf
 network 10.10.10.0/24 area 0
 network 192.168.200.0/24 area 0
 end
write memory
```

Настройки на роутере Cisco:
```
configure terminal
 router ospf 1
 network 10.10.10.0 0.0.0.3 area 0
 network 192.168.100.0 0.0.0.255 area 0
 end
write memory
```

Проверим доступность не directly connected сетей.  
Со стороны Linux-роутера:
```
linuxrouter# sh ip ospf neighbor
Neighbor ID     Pri State           Dead Time Address         Interface                        RXmtL RqstL DBsmL
192.168.100.1     1 Full/DROther      37.158s 10.10.10.1      vti0:10.10.10.2                      0     0     0
```
```
linuxrouter# show ip route ospf
...
O   10.10.10.0/24 [110/10] is directly connected, vti0, weight 1, 00:17:04
O   10.10.10.0/30 [110/1010] via 10.10.10.1, vti0 inactive, weight 1, 00:15:12
O>* 192.168.100.0/24 [110/20] via 10.10.10.1, vti0, weight 1, 00:15:03
O   192.168.200.0/24 [110/1] is directly connected, ens4, weight 1, 00:16:53
```
```
linuxrouter# ping 192.168.100.1
PING 192.168.100.1 (192.168.100.1) 56(84) bytes of data.
64 bytes from 192.168.100.1: icmp_seq=1 ttl=255 time=2.95 ms
64 bytes from 192.168.100.1: icmp_seq=2 ttl=255 time=0.927 ms
64 bytes from 192.168.100.1: icmp_seq=3 ttl=255 time=1.46 ms
--- 192.168.100.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.927/1.779/2.954/0.858 ms
```

Со стороны Cisco:
```
CiscoRouter#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.200.1     0   FULL/  -        00:00:32    10.10.10.2      Tunnel0
```
```
CiscoRouter#show ip route ospf
...
      10.0.0.0/8 is variably subnetted, 3 subnets, 3 masks
O        10.10.10.0/24 [110/1010] via 10.10.10.2, 00:18:32, Tunnel0
O     192.168.200.0/24 [110/1001] via 10.10.10.2, 00:18:32, Tunnel0
```
```
CiscoRouter#ping 192.168.200.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.200.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/5/7 ms
```

## Проверка доступности между ПК

В качестве образа для ПК будет использоваться встроенный в EVE-NG образ Virtual PC.  

Настроим ПК в сети 192.168.100.0/24.  
```
VPCS> set pcname PC-1
PC-1> ip 192.168.100.100/24 192.168.100.1
PC-1> save
```

Такая же конфигурация для ПК в сети 192.168.200.0/24.
```
VPCS> set pcname PC-2
PC-2> ip 192.168.200.200/24 192.168.200.1
PC-2> save
```

Проверим доступность с PC-1 до PC-2 и наоборот, как видно все работает.
```
PC-1> ping 192.168.200.200
84 bytes from 192.168.200.200 icmp_seq=1 ttl=62 time=2.765 ms
84 bytes from 192.168.200.200 icmp_seq=2 ttl=62 time=1.810 ms
84 bytes from 192.168.200.200 icmp_seq=3 ttl=62 time=2.045 ms
84 bytes from 192.168.200.200 icmp_seq=4 ttl=62 time=2.146 ms
84 bytes from 192.168.200.200 icmp_seq=5 ttl=62 time=2.854 ms

PC-1> trace 192.168.200.200
trace to 192.168.200.200, 8 hops max, press Ctrl+C to stop
 1   192.168.100.1   0.606 ms  0.350 ms  0.871 ms
 2   10.10.10.2   2.068 ms  0.916 ms  0.961 ms
 3   *192.168.200.200   1.975 ms (ICMP type:3, code:3, Destination port unreachable
```

```
PC-2> ping 192.168.100.100
84 bytes from 192.168.100.100 icmp_seq=1 ttl=62 time=1.832 ms
84 bytes from 192.168.100.100 icmp_seq=2 ttl=62 time=2.061 ms
84 bytes from 192.168.100.100 icmp_seq=3 ttl=62 time=2.307 ms
84 bytes from 192.168.100.100 icmp_seq=4 ttl=62 time=1.803 ms
84 bytes from 192.168.100.100 icmp_seq=5 ttl=62 time=2.827 ms

PC-2> trace 192.168.100.100
trace to 192.168.100.100, 8 hops max, press Ctrl+C to stop
 1   192.168.200.1   0.639 ms  0.237 ms  0.218 ms
 2   10.10.10.1   1.875 ms  1.319 ms  1.065 ms
 3   *192.168.100.100   1.133 ms (ICMP type:3, code:3, Destination port unreachable
```
