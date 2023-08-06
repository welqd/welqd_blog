---
title: "Proxmox. Организация доступа лабы в EVE-NG в локальную сеть и/или Интернет"
date: 2023-04-16T12:11:11+03:00
author: Vadim Aleshin
tags: ['networking', 'linux', 'eve-ng', 'proxmox']
coverImage: /img/eve-ng-proxmox-topology.png
---

Нередко необходимо иметь доступ к устройствам внутри лабораторной EVE-NG из домашней сети. К примеру: для использования локального DHCP или DNS сервера, либо для тестирования автоматизации/скриптов с рабочего ПК.  

Существует несколько вариантов добиться данной цели, рассмотрю, как по мне, наиболее удобный.  

###  Топология локальной сети  

Все виртуальные девайсы в EVE-NG будут в отдельной VLAN'е 69 и подсети 10.69.69.0/24, для удобства.  
Идея в том, чтобы "дотянуть" VLAN 69 до виртуалки с EVE.  
В качестве VE у меня выступает Proxmox, но в VMware логика такая же.  

Рассмотрим топологию моих устройств и остановимся на каждом поподробнее.  

![Home Network Topology](/img/eve-ng-proxmox-topology.png)

a) На роутере (в моем случае это просто мини ПК с Linux) необходимо создать VLAN 69, добавить его tagged на порт, смотрящий в сторону свитча и на VLAN-интерфейс повесить IP 10.69.69.1/24.
Необходимо также убедиться, что работает Inter VLAN Routing (включен ip forwarding и в iptables/nftables не запрещено общение между подсетями).  
В большинстве роутеров будет работать по умолчанию.  

b) На коммутаторе, необходимо просто добавить VLAN 69 tagged на порт в сторону роутера и в сторону Proxmoxa.  

c) На Proxmox необходимо включить VLAN aware на bridge интерфейсе и добавить ещё один сетевой интерфейс на виртуальную машину, с VLAN 69, как на скриншотах ниже.  

![VLAN aware bridge](/img/proxmox-vlan-aware-bridge.png)
![EVE-NG second NIC](/img/proxmox-second-nic-eve-ng.png)

Данный интерфейс, смотрит в сторону VM как untagged, т.е. на устройствах в EVE-NG VLAN 69 уже создавать будет не нужно.  

### Настройка со стороны EVE-NG

После того, как добавили сетевую карту на VM, необходимо проверить какому "cloud" интерфейсу она соответствует.  
Берём MAC-адрес с предыдущего скриншота, подключаемся к VM и смотрим.  
```
root@eve-ng:~# ip add | grep -B 1 -i 02:1F:48:AB:13:A2
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master pnet1 state UP group default qlen 1000
   link/ether 02:1f:48:ab:13:a2 brd ff:ff:ff:ff:ff:ff

5: pnet1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
   link/ether 02:1f:48:ab:13:a2 brd ff:ff:ff:ff:ff:ff
```
pnet1 - это Cloud 1 в интерфейсе EVE-NG.  

Теперь нам необходимо добавить Network в лабораторной EVE-NG (правой кнопкой -> Network) и выбрать Type Cloud 1.  

![Home Network Topology](/img/eve-ng-to-outside-cloud1.png)

Следующим шагом добавляем любой неуправляемый свитч (на изображении ниже это OUTSIDE_SWITCH) и соединяем его с ранее созданным облаком. К этому свитчу будем подключать все устройства, доступ к которым необходим из-за пределов лабораторной.  
Также необходим будет маршрут в локальную сеть (10.222.21.0/24) или default route (для доступа ещё и в Интернет) на каждом из устройств в EVE-NG через шлюз 10.69.69.1.    

![Home Network Topology](/img/eve-ng-to-outside-topology.png)

Настройка роутера R2:  
```
Router>enable
Router#conf t
Router(config)#interface ethernet 0/0
Router(config-if)#no shutdown
Router(config-if)#ip address 10.69.69.10
Router(config-if)#ip address 10.69.69.10 255.255.255.0
Router(config-if)#exit
Router(config)#ip route 10.222.21.0 255.255.255.0 10.69.69.1
Router#write memory
```

Настройка коммутатора (на схеме имя Switch):  
```
Switch(config)#vlan 123
Switch(config-vlan)#exit
Switch(config)#interface vlan 123
Switch(config-if)#ip address 10.69.69.11 255.255.255.0
Switch(config-if)#no shutdown
Switch(config-if)#exit
Switch(config)#int gigabitEthernet 0/0
Switch(config-if)#no shutdown
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 123
Switch(config-if)#exit
Switch(config)#ip route 10.222.21.0 255.255.255.0 10.69.69.1
Switch(config)#end
Switch(config)#write memory
```

Проверим доступность с рабочего ПК. Ping, к примеру, до коммутатора в EVE-NG:
```
ping -c 2 10.69.69.11 -I 10.222.21.138
PING 10.69.69.11 (10.69.69.11) from 10.222.21.138 : 56(84) bytes of data.
64 bytes from 10.69.69.11: icmp_seq=1 ttl=254 time=2.15 ms
64 bytes from 10.69.69.11: icmp_seq=2 ttl=254 time=2.47 ms

--- 10.69.69.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 2.153/2.312/2.472/0.159 ms
```
Итого: для доступа к любому устройству из лабы, необходимо его подключить в коммутатор управления (на схеме OUTSIDE_SWITCH), назначить ему адрес из сети 10.69.69.0/24 и прописать маршрут в локальную сеть (в моем случае 10.222.21.0/24) через шлюз 10.69.69.1.  

### Способ с подключением только одного устройства к Cloud  

Можно не подключать каждое из устройств к коммутатору управления, как в примере выше, а подключить, к примеру, роутер R2 напрямую к Cloud 1 и на остальных устройствах использовать уже сети, которые захочется.  
В таком случае необходим будет маршрут на домашнем роутере в сети устройств из лабораторной.  
Создадим на R2 loopback интерфейс с адресом 10.2.2.2/24.
```
Router>enable
Router#conf t
Router(config)#int lo0
Router(config-if)#ip add 10.2.2.2 255.255.255.0
Router(config-if)#end
Router#wr mem
```
Проверим доступность данного адреса с ПК (на котором ip 10.222.21.138).  
```
ping -c 2 10.2.2.2 -I 10.222.21.138
PING 10.2.2.2 (10.2.2.2) from 10.222.21.138 : 56(84) bytes of data.

--- 10.2.2.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1006ms
```
Как и предполагалось, пинг не прошёл, т.к. роутер ничего не знает о сети 10.2.2.0/24.  
Подключимся к нему и добавим маршрут.  
```
─❯ ssh 10.222.21.1
root@router-box:~# ip route add 10.2.2.0/24 via 10.69.69.1
```
Теперь с ПК проверим доступность:  
```
ping -c 2 10.2.2.2 -I 10.222.21.138
PING 10.2.2.2 (10.2.2.2) from 10.222.21.138 : 56(84) bytes of data.
64 bytes from 10.2.2.2: icmp_seq=1 ttl=254 time=1.38 ms
64 bytes from 10.2.2.2: icmp_seq=2 ttl=254 time=1.31 ms

--- 10.2.2.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.314/1.346/1.379/0.032 ms
```
Как видно все работает.
