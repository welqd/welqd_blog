---
title: "Настройка роутера на Linux. Часть 1. Установка ПО и настройка интерфейсов"
date: 2023-08-05T20:16:17+03:00
lastmod: 2024-01-14T22:19:37+03:00
author: Vadim Aleshin
tags: ["networking", "linux", "linux router"]
---

## Предисловие

Это первая статья из цикла,
в котором попытаемся превратить ПК в полнофункциональный роутер.  
На этой странице, ниже по тексту, будет рассмотрена
базовая конфигурация интерфейсов (используя systemd-networkd).  
В последующих публикациях совершим базовую настройку:

1. [**nftables**, в качестве firewall'а;]({{< ref "/posts/2024-01-14-linux_router_p2_nftables.md" >}})

2. **DHCP (kea)** и **DNS (bind9)** серверов;
3. **Wireguard**, для организации Remote access и Site-to-Site VPN'ов;
4. **FRR**, для создания простого сценария с установлением BGP-сессии
   и перенаправления интересующих нас префиксов в wg-туннель.

Рассматривать настройку WiFi Access Point не будем,
т.к. у меня нет подходящих компонентов.

## Выбор аппаратной платформы

Для роутера подойдёт любой компьютер с двумя сетевыми интерфейсами,
на который можно установить Linux.  
Я использую мини ПК с Aliexpress.
За более чем год работы в режиме 24/7 каких-либо проблем не нашёл.  
Отличными кандидатами являются **Orange Pi 5 plus** на ARM-архитектуре,
либо **VisionFive 2** на RISC-V (обе платы имеют по два интерфейса).
На обе архитектуры доступен Debian,
так что особых различий от x64_86 быть не должно, хотя я этого не проверял.

## Выбор Linux-дистрибутива и почему systemd-networkd?

В принципе, подойдёт любой дистрибутив
(так как в этой статье будет использоваться **systemd-networkd**
для назначения адресов на интерфейсы
, то дистрибутив должен быть с ним),
но стоит обратить внимание на наличие драйверов для сетевых карт,
в моем случае необходимый драйвер появился в ядре 5.7.  
В данной статье будет использоваться Debian 12.  
Почему **systemd-networkd**?  
Удобно организовывать конфигурационные файлы,
не нужно ставить дополнительных пакетов для работы с интерфейсами,
таких как bridge-utils, vlan, даже wireguard встроен в ядро
и пакет нужен только для генерации ключей.  
Также конфигурация легко переносится на любой дистрибутив,
вне зависимости от используемого метода для сетевых настроек.
Да и просто хотелось с ним немного больше разобраться.

## Установка пакетов

Необходимо определиться с ПО,
которое будет использоваться для обеспечения всех необходим функций.  
В этой серии, как было указано в предисловии,
будем использовать nftables (firewall), kea (dhcp сервер), bind (dns сервер),
wireguard (все, относящееся к vpn), frr (для динамической машрутизации).  
Также добавим несколько пакетов для диагностики (tcpdump, ethtool и т.д.)

Итого, для работы с сетью:

```
apt install nftables bind9 bind9-doc kea-dhcp4-server kea-doc wireguard frr tcpdump conntrack ethtool whois nmap ssh snmp snmpd
```

Для удобства администрирования ещё поставим:

```
apt install vim htop lm-sensors curl
```

## Переименование интефрейсов

В моем случае один из сетевых интерфейсов именовался eno1,
что отличалось от схемы остальных интерфейсов.

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:e2:69:53:be:35 brd ff:ff:ff:ff:ff:ff
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:e2:69:53:be:36 brd ff:ff:ff:ff:ff:ff
4: eno1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 00:e2:69:53:be:37 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
5: enp4s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 00:e2:69:53:be:38 brd ff:ff:ff:ff:ff:ff
```

Имена назначаются по приоритету из файла
**/usr/lib/systemd/network/99-default.link**, который содержит данные в секции Link

```
[Link]
NamePolicy=keep kernel database onboard slot path
```

Проверить, какие значение имеет конкретный интерфейс можно командой:

```
udevadm test-builtin net_id /sys/class/net/eno1 2>/dev/null
```

В моем случае на 3-ем порту есть значение **onboard**, что является приоритетным
значением относительно **path**, при этом на других портах оно отсутствует.

```
ID_NET_NAMING_SCHEME=v247
ID_NET_NAME_MAC=enx00e26953be37
ID_NET_NAME_ONBOARD=eno1
ID_NET_LABEL_ONBOARD=Onboard - RTK Ethernet
ID_NET_NAME_PATH=enp3s0
```

Для изменения имени:

1. Смотрим MAC-адрес интересующего нас интерфейса:

```
ip link show
4: enp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master br0 state DOWN mode DEFAULT group default qlen 1000
    link/ether 00:e2:69:53:be:37 brd ff:ff:ff:ff:ff:ff
```

2. Создадим файл в **/etc/systemd/network/** и назовем его **90-enp3s0.link**
   (для того, чтобы systemd-networkd вообще посмотрел внутрь файла, его имя должно
   стоять ранее в алфавитном порядке, чем стандартное имя 99-default.link).  
    В секции Match указываем MAC, в секции Link - имя.

```
[Match]
MACAddress=00:e2:69:53:be:37
[Link]
Name=enp3s0
```

После чего необходимо обновить образ
и перезагрузить систему **update-initramfs -u && reboot**

## Включение маршрутизации между интерфейсами

Для использования ОС в качестве роутера,
нам необходимо включить маршрутизацию между интерфейсами.
В файле **/etc/sysctl.conf** находим строку
**net.ipv4.ip_forward=1** и раскомментируем её.

## Настройка интерфейсов

Т.к. мы используем **systemd-networkd**, то все настройки будут храниться
в текстовых файлах в директории **/etc/systemd/network**
Для настройки используются 3 типа файлов:

1. .network - применения конфигурации на интерфейсах
2. .netdev - создания виртуальных сетевых интерфейсов
3. .link - в данный файл смотрит udev, при обнаружении интерфейсов

В моем примере будет использоваться 3 порта:

1. enp1s0 - смотрит в сторону провайдера
2. enp2s0, enp3s0 - для создания bridge интерфейса для подключения LAN-сегмента.

Имена файлов можно использовать произвольные, только необходимо помнить,
что конфигурация из них читается в алфавитном порядке,
т.е., к примеру, чтобы настроить IP-адрес на bridge интерфейсе,
необходимо чтобы к моменту чтения конфигурационного .network файла,
данный интерфейс уже был создан .netdev файлом.

По умолчанию Debian использует файлы из директории
**/etc/network** для конфигурации сети.
Т.к. мы будем использовать systemd-networkd,
то первым делом переименуем стандартный конфигурационный файл
и отключим соответствующий сервис:

> **_NOTE:_** Если данное действие делать удалённо, то, очевидно,
доступ к устройству пропадёт.
Поэтому либо сначала полностью создаём все конфигурационные systemd-networkd файлы,
после чего отключаем networking, либо необходим локальный доступ.

```
mv /etc/network/interfaces /etc/network/interfaces.save
systemctl disable networking.service
systemctl stop networking.service
```

### Настройка WAN-интерфейса

Мне провайдер предоставляет настройки по DHCP (IPoE с привязкой по MAC-адресу),
конфигурационный 30-enp1s0.network файл будет выглядеть примерно так
(т.к. ipv6 в данной статье не используется,
то запретим создание Link Local адресов на интерфейсах):

```
[Match]
Name = enp1s0

[Network]
DHCP = ipv4
LinkLocalAddressing = no
IPv6AcceptRA = no
KeepConfiguration = dhcp
IgnoreCarrierLoss = yes

[DHCPv4]
SendRelease = no
```

Для более подробной информации по каждому параметру:

```
man systemd.network
```

### Настройка BRIDGE, в качестве LAN-интерфейса

В качестве интерфейса, смотрящего в локальную сеть
будет использоваться Bridge-интерфейс, объединяющий порты enp2s0 и enp3s0.  
Для создания бриджа нам потребуется 3 файла:

1. 20-bind.network, где указываем интерфейсы, которые будут использоваться в Bridge

```
[Match]
Name=enp[2,3]s0

[Network]
Bridge=br0
```

2. 20-bridge.netdev, в данном файле создаем уже сам интерфейс

```
[NetDev]
Name=br0
Kind=bridge
```

3. 20-bridge.network, где указываем сетевые настройки интерфейса
(в файле настроен ещё vlan69, его создание будет в следующем пункте)

```
[Match]
Name = br0

[Network]
Address = 10.222.21.1/24
LinkLocalAddressing = no
VLAN=vlan69
```

### Добавление VLAN-интерфейса

Также создадим VLAN-интерфейс и добавим на ранее созданнный br0-интерфейс.
Для этого необходимы 2 файла:

1. 69-vlan69.netdev, для создания виртуального интерфейса

```
[NetDev]
Name=vlan69
Kind=vlan

[VLAN]
Id=69
```

2. 69-vlan69.network, для настроек на ранее созданном интерфейсе

```
[Match]
Name=vlan69
Type=vlan

[Network]
Description="proxmox ve"
LinkLocalAddressing = no

[Address]
Address=10.69.69.1/24
```

## Заключение

На данном этапе настроены интерфейсы для WAN и LAN подключений,
но роутер ещё не имеет доступ в интернет,
т.к. не настроен NAT,
настройка которого рассматривается в [следующей статье]({{< ref "/posts/2024-01-14-linux_router_p2_nftables.md" >}})  
Также отсутствуют локальные DNS и DHCP серверы,
настройка которых рассматривается в третьей части.
