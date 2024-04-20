---
title: "Настройка роутера на Linux. Часть 2. NFTables: Firewall и NAT"
date: 2024-01-14T21:13:01+03:00
author: Vadim Aleshin
tags: ["networking", "linux", "linux router"]
---

В [первой части]({{< ref "/posts/2023-08-06-linux_router_p1_basics.md" >}})
была рассмотрена настройка интерфейсов, теперь донастроим Firewall и NAT,
чтобы устройство могло полноценно работать.

Если сразу нужен готовый конфиг - он в конце статьи.

Рассмотрим довольно простую конфигурацию, которую потом можно будет расширить
при необходимости.  
Для реализации функционала Firewall будем использовать nftables.

### Описание работы Nftables/Netfilter

**Nftables** - это преемник iptables
(и других \*tables, таких ip6tables, arptables, ebtables),
объединяющий весь их функционал в одно приложение.  
Nftables, как и iptables, "под капотом" использует Netfilter.  
Ниже представлена диаграмма прохождения пакета Netfilter с [официальной страницы nftables.org](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)
![nf-hooks](/img/2024-01-14-linux_router_p2_nftables/nf-hooks.png)

Простыми словами (вероятно, что с неточностями) то,
как пакет проходит в **Ip Layer** (зелёная часть диаграммы выше):

В ядре представлены серия "хуков", через которые может проходить пакет.  
**Prerouting** - хук, в который попадает пакет при поступлении на устройство.
На этом этапе проверяется, адресован ли пакет на какой-либо локальный процесс
системы (к примеру, на открытый сокет DNS-сервера).  
Если да, то пакет в таком случае попадает в хук **Input**
и адресуется этому локальному процессу.  
Если же пакет **не предназначен** для локальной системы,
а должен быть смаршрутизирован, то он передается хуку **Forward**.  
И в заключении пакет попадает в хук **Postrouting** перед тем,
как он покинет устройство.  
Если пакет сгенерирован **локально** на роутере,
то сначала он должен пройти хук **Output**,
после чего попасть в хук **Postrouting**, после чего уже покинуть устройство.

### Пример использования cli утилиты nft

Конфигурировать nftables можно с помощью встроенной команды nft, к примеру:

1. Создать таблицу: **ntf add table ip FILTER**

2. Добавить цепочку:
   **nft add chain ip INPUT input { type filter hook input priority 0\; }**

3. Посмотреть текущие правила и счётчики: **nft list ruleset**

Подробный синтаксис в [официальной документации](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)

Лично мне, такой способ кажется неудобным,
поэтому настройки будем производить сразу в конфигурационном файле **/etc/nftables.conf**

### Настройка правил в файле /etc/nftables.conf

Далее по пунктам будут выделены "логические" куски конфигурации,
которые в итоге составят итоговое содержание файла /etc/nftables.conf.

1. Файл должен начинаться с исполняемой программы nft и очистки правил:

```text
#!/usr/sbin/nft -f
flush ruleset
```

2. Определим переменные с необходимыми интерфейсами,
   для удобства дальнейшей настройки.  
    По этому принципу можно организовать Zone Based Firewall,
   если несколько интерфейсов принадлежат одной зоне безопасности).

```text
define WAN_INT = enp1s0
define LAN_INT = br0
define VLAN69 = vlan69
```

3. Определим переменную для немашрутизируемых глобально сетей,
   чтобы явно заблокировать поступающие на WAN-интерфейс пакеты.

```text
define BOGONS4 = {
0.0.0.0/8,
10.0.0.0/8,
10.64.0.0/10,
127.0.0.0/8,
127.0.53.53,
169.254.0.0/16,
172.16.0.0/12,
192.0.0.0/24,
192.0.2.0/24,
192.168.0.0/16,
198.18.0.0/15,
198.51.100.0/24,
203.0.113.0/24,
224.0.0.0/4,
240.0.0.0/4,
255.255.255.255/32,
}
```

4. Создадим таблицу ip с именем FILTER, в которой укажем цепочки:

- FROM_WAN; в данной цепочке отбрасываем ранее определённые BOGONS-сети.  
  Также, все пакеты, пришедшие на WAN-интерфейс, будем перенаправлять сюда,
  для удобства управления (это правило будет далее в цепочке INPUT).

- FROM_LAN: в данной цепочке разрешаем необходимые сервисы
  (в данном случае: ssh, dns, dhcp, bgp, snmp и zabbix-agent).
  Как и в предыдущем случае, сюда перенаправляем все пакеты из хука input,
  пришедшие с LAN-интерфейсов

```text
table ip FILTER {
  chain FROM_WAN {
    ip saddr { $BOGONS4 } counter drop
    counter drop
 }
  chain FROM_LAN {
    tcp dport { 22, 53, 179, 10050 } counter accept
    udp dport { 67, 53, 161 } counter accept
    counter drop
 }
```

5. Далее настройка цепочки input. Сразу приведу всю конфигурацию,
   а чуть ниже рассмотрим каждую строку отдельно.

```text
chain INPUT {
    type filter hook input priority 0; policy accept;
    ct state established,related counter accept
    ct state invalid counter drop
    icmp type echo-request meta length 1529-65535 counter drop
    ip protocol icmp limit rate 10/minute counter accept
    iifname lo ip daddr != 127.0.0.0/8 counter drop
    iifname lo counter accept
    iifname $WAN_INT counter jump FROM_WAN
    iifname $LAN_INT counter jump FROM_LAN
    counter drop
}
```

- Создаём цепочку с именем INPUT
- Указываем тип: filter, hook: input и правило по умолчанию: accept
- Принимаем пакеты, принадлежащие established и related соединениям:
  - **established** - это состояние, которое принимает пакеты,
    принадлежащие уже существующему соединению,
    т.е. пакеты, адресованные обратно инициатору соединения будут приниматься
    автоматически, что делает наше устройство **statefull**-firewall'ом.
  - **related** - это состояние, в котором пакет инициирует новое соединение,
    которое ассоциировано с уже существующим established-соединением.
    Как указано в man к iptables - это может быть, к примеру, FTP данные
    (т.к. каждый FTP-клиент становится также сервером для передачи данных),
    либо ICMP error.
- Отбрасываем пакеты с состоянием invalid
  - **invalid** - пакет, который не может быть идентифицирован или не имеет
    какого-либо состояния
- Все пакеты размером более 1528 - отбрасываем
- Ограничиваем ICMP-пакеты 10тью в минуту
  (тут надо быть осторожным, если, к примеру, мониторится по ICMP внешний интерфейс,
  то очевидно, что будут "потери").
- Дропаем входящие пакеты на lo-интерфейс,
  адрес назначения которых не принадлежит сети 127.0.0.0/8
- Принимаем пакеты на lo-интерфейс
- Переадресовываем все пакеты, пришедшие с WAN-интерфейса в цепочку FROM_WAN
  (правила для которой указаны ранее по тексту)
- Переадресовываем все пакеты, пришедшие с LAN-интерфейса в цепочку FROM_LAN
  (правила для этой цепочки также выше по тексту)
- Все остальное отбрасываем
  (т.к. ранее по умолчанию было включено правило accept).
  Это сделано для того, чтобы логировать попавшие пакеты (counter в конфиге).

6. Настройки цепочки forward.  
   Принимаем established и related соединения, отбрасываем invalid,
   как в цепочке input.  
   Разрешаем пакеты на интерфейсах: LAN_INT (br0), VLAN69 (vlan69)  
   Остальное отбрасываем.

```text
chain FORWARD {
    type filter hook forward priority 0; policy accept;
    ct state established,related counter accept
    ct state invalid counter drop
    iifname { lo, $LAN_INT, $VLAN69 } counter accept
    counter drop
}
```

### Настройка NAT

Последним действием будет настройка NAT. Тут все просто:

- создаём таблицу с именем NAT (или любым другим)
- создаём цепочку с хуком postrouting и policy accept
- указываем oif (outgoing interface) - WAN_INT и действие masquerade

```text
table ip NAT {
  chain POSTROUTING {
    type nat hook postrouting priority 0; policy accept;
    oif { $WAN_INT } masquerade
  }
}
```

### Итоговая конфигурация

Просто приведу итоговый конфиг, получившийся из ранее описанных действий.

```text
#!/usr/sbin/nft -f

flush ruleset

define WAN_INT = enp1s0
define LAN_INT = br0
define VLAN69 = vlan69

define BOGONS4 = {
0.0.0.0/8,
10.0.0.0/8,
10.64.0.0/10,
127.0.0.0/8,
127.0.53.53,
169.254.0.0/16,
172.16.0.0/12,
192.0.0.0/24,
192.0.2.0/24,
192.168.0.0/16,
198.18.0.0/15,
198.51.100.0/24,
203.0.113.0/24,
224.0.0.0/4,
240.0.0.0/4,
255.255.255.255/32
}

table ip FILTER {
    chain FROM_WAN {
        ip saddr { $BOGONS4 } counter drop
        counter drop
    }
    chain FROM_LAN {
        tcp dport { 22, 53, 179, 10050 } counter accept
        udp dport { 67, 53, 161 } counter accept
        counter drop
    }
    chain INPUT {
        type filter hook input priority 0; policy accept;
        ct state established,related counter accept
        ct state invalid counter drop
        icmp type echo-request meta length 1529-65535 counter drop
        ip protocol icmp limit rate 10/minute counter accept
        iifname lo ip daddr != 127.0.0.0/8 counter drop
        iifname lo counter accept
        iifname $WAN_INT counter jump FROM_WAN
        iifname $LAN_INT counter jump FROM_LAN
        counter drop
    }
    chain FORWARD {
        type filter hook forward priority 0; policy accept;
        ct state established,related counter accept
        ct state invalid counter drop
        iifname { lo, $LAN_INT, $VLAN69 } counter accept
        counter drop
    }
}

table ip NAT {
    chain POSTROUTING {
        type nat hook postrouting priority 0; policy accept;
        oif { $WAN_INT } masquerade
    }
}
```

Ну и в завершении включаем (если не был включен) или перезагружаем сервис nftables

```text
systemctl enable --now nftables.service
systemctl restart nftables.service
```

Если ошибок нет, то сервис будет в состоянии active.

```text
● nftables.service - nftables
     Loaded: loaded (/lib/systemd/system/nftables.service; enabled; preset: enabled)
     Active: active (exited) since Sun 2024-01-14 15:04:32 MSK; 6h ago
       Docs: man:nft(8)
             http://wiki.nftables.org
    Process: 5001 ExecStart=/usr/sbin/nft -f /etc/nftables.conf (code=exited, status=0/SUCCESS)
   Main PID: 5001 (code=exited, status=0/SUCCESS)
        CPU: 18ms
```
