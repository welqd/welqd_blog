---
title: "Образ роутера на Linux для EVE-NG"
date: 2022-09-15T14:46:00+03:00
author: Vadim Aleshin
tags: ['networking', 'linux', 'eve-ng']
---

В [предыдущей]({{< ref "/posts/2022-09-13-eve-ng-arch-i3.md" >}}) публикации был установлен образ EVE-NG и настроена базовая конфигурация. Теперь создадим образ Linux-роутера на основе Debian 11 и добавим его в EVE-NG.   

## Создание виртуальной машины

[Скачиваем](https://cdimage.debian.org/debian-cd/current/amd64/bt-cd/) образ с официального сайта.  
Создаем виртуальную машину, для этих целей будет использоваться VMware Workstation. 
В процессе настройки ВМ на этапе указания объема диска **обязательно** выбрать **Store virtual disk as a single file**.  
Т.к. необходим наиболее минимальный образ, то объем диска указываем 3-4 Гб, этого достаточно.  
Также, для удобства, на заключительном этапе установки делаем настройку сетевой карты мостом, для этого выбираем **Customize Hardware...**, выбираем **Network adapter** -> **Bridged**, как на изображении ниже.  

![Настройка bridge](/img/vmware-bridge.png)

После этого запускаем ВМ и устанавливаем операционную систему.  
На этапе установки стоит обратить внимание на то, что графическое окружение устанавливать **не нужно**, поэтому на шаге **software selection** оставляем отмеченными только пункты **SSH server** и **standart system utilities**.

После установки ОС добавим возможность подключения по ssh, для удобства конфигурации. Для этого в файле **/etc/ssh/ssdh_config** в секцию **Authentication** добавим строку **PermitRootLogin yes**, перезапустим сервис ssh **systemctl restart ssh**.  

## Установка ПО на виртуальную машину

Подключаемся по ssh к ВМ для установки и настройки необходимых сервисов:
1. Обновляем ОС
```
apt update && apt upgrade
```
2. Для дальнейшего подключения к роутеру в EVE-NG необходимо:
- Установить telnet 
```
apt install telnetd
```
- Проверить, что сервис включен и запущен (после установки он самостоятельно включается):
```
systemctl status inetd.service
```
```
● inetd.service - Internet superserver
Loaded: loaded (/lib/systemd/system/inetd.service; enabled; vendor preset: enabled)
Active: active (running) since Thu 2022-09-15 17:11:40 MSK; 1min 4s ago
```
- Также необходимо включить доступ по консольному интерфейсу (т.к. EVE-NG использует его, при отсутствии VNC):
```
systemctl start serial-getty@ttyS0.service  
systemctl enable serial-getty@ttyS0.service
```
3. Установим программы для работы с сетью:
- [FRR](https://frrouting.org/) - набор утилит для работы с протоколами маршрутизации
```
apt install frr
```
- Strongswan - для работы с IPSec
```
apt install strongswan
```
- Программы для диагностики:
```
apt install traceroute tcpdump dnsutils
```
3. Выключаем виртуальную машину.

## Добавление образа в EVE-NG[^1]

Созданный ранее образ загружаем на виртуальную машину с EVE-NG, для этого:
1. Заходим по ssh на сервер с EVE-NG и создаем там директорию для образа:
```
mkdir /opt/unetlab/addons/qemu/linuxrouter-1.0
```
Необходимо учитывать [схему именования qemu устройств](https://www.eve-ng.net/index.php/documentation/qemu-image-namings/), сначала название папки с образом, а через дефис название образа и версия, либо просто версия. Это имя будет использоваться в шаблоне, для создания устройства.  

2. На машине, где создавали образ, переходим в директорию с ним и загружаем .vmdk файл на сервер:
```
scp linuxrouter.vmdk root@192.168.22.71:/opt/unetlab/addons/qemu/linuxrouter-1.0/
```
3. На сервере с EVE-NG перейдем в директорию, куда копировали образ, переконвертируем его в формат .qcow2, после чего удалим .vmdk файл:
```
cd /opt/unetlab/addons/qemu/linuxrouter-1.0/  
/opt/qemu/bin/qemu-img convert -f vmdk -O qcow2 linuxrouter.vmdk hda.qcow2  
rm linuxrouter.vmdk
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
4. Создадим шаблон для образа, для этого:
- Перейдем в директорию /opt/unetlab/html/templates/{тип процессора}
```
cd /opt/unetlab/html/templates/amd
```
- Скопируем существующий шаблон linux.yml в новый linuxrouter.yml
```
cp linux.yml linuxrouter.yml
```
- Отредактируем файл linuxrouter.yml и приведем его к следующему виду: 
```
---
type: qemu
description: LinuxRouter
name: LinuxRouter
cpulimit: 1
icon: Router.png
cpu: 1
ram: 1024
ethernet: 4
console: telnet
shutdown: 1
qemu_arch: x86_64
qemu_nic: virtio-net-pci
qemu_options: -machine type=pc,accel=kvm -serial mon:stdio -nographic -boot order=c
...
```

Здесь мы указали: описание, имя, количество CPU и RAM, количество портов, telnet в качестве консоли и в параметрах qemu использование serial интерфейса

## Проверка работы образа

Теперь при добавлении новой Node появляется выбор образа  
![Новая Node](/img/eve-ng-add-linuxrouter.png)

Также видно, что параметры по умолчанию используются те, которые задавали ранее в шаблоне
![Параметры](/img/eve-ng-linuxrouter-template.png)

Соберем простую топологию, с первое устройство будет роутер Cisco, второе - роутер на Linux
![Топология](/img/eve-ng-linuxrouter-topology.png)

Настроим ip-адреса на обоих устройствах и сделаем ping друг до друга  

1. Конфиг Cisco
```
Router>enable
Router#conf t
Router(config)#hostname CiscoRouter
CiscoRouter(config)#int e0/0
CiscoRouter(config-if)#no sh
CiscoRouter(config-if)#ip add 1.1.1.1 255.255.255.0
```
2. Конфиг роутера на Linux
```
root@linuxrouter:~# vtysh
linuxrouter# configure terminal
linuxrouter(config)# interface ens3
linuxrouter(config-if)# ip address 1.1.1.2/24
```
3. Проверим ping с двух сторон
```
CiscoRouter#ping 1.1.1.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
```
linuxrouter# ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=255 time=0.940 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=255 time=0.655 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=255 time=0.954 ms
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 0.655/0.849/0.954/0.137 ms
```

Как видим образ работает. В [следующей статье]({{< ref "/posts/2022-09-17-linux-cisco-vti-ipsec.md" >}}) установим ipsec туннель между Cisco и Linux роутером и настроим OSPF.

[^1]: В качестве основы для данной публикации использовалась [данная](https://www.brianlinkletter.com/2017/03/build-custom-linux-router-image-unetlab-eve-ng-network-emulators/) статья.
