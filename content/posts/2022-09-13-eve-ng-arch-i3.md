---
title: "EVE-NG установка + интеграция в Arch Linux с i3wm"
date: 2022-09-13T20:16:17+03:00
author: Vadim Aleshin
tags: ['networking', 'linux', 'eve-ng', 'i3']
---

## Установка EVE-NG в VMware

Предполагается, что уже есть установленная VMware workstation, для Arch Linux можно установить из [AUR](https://aur.archlinux.org/packages/vmware-workstation).  
Для установки EVE Community Edition Version:  
1. Скачиваем архив с образом с официального [сайта](https://www.eve-ng.net/index.php/download/)
2. Распаковываем архив и добавляем в VMware workstation (File -> Open -> EVE-COM-5.ovf)
3. Запускаем VM и входим в ОС под пользователем root, пароль eve
4. Задаем первичные параметры в появившемся диалоге настроек (root пароль, ip адрес и т.д.)

После этого можно зайти в web-интерфейс по ip-адресу (в этой статье примеры будут с адресом 192.168.22.71), стандартный логин - admin, пароль - eve.  

### Добавление образов

В EVE-NG можно использовать разные типы образов: dynamips, iol, qemu.  
Для примера будем использовать образы Cisco (iol), а после добавим [custom linux образ]({{< ref "/posts/2022-09-15-eve-ng-linux-iso.md" >}}).  
IOL-образы необходимо добавлять по пути: **/opt/unetlab/addons/iol/bin/**

1. Перейдем в директорию на локальной машине, где располагаются образы[^1].
[^1]: Необходимо либо использовать официальные образы, к примеру из Cisco CML, либо "нагуглить" не совсем официальные 
2. Используя scp добавим их на виртуальную машину с EVE-NG:  
```
scp * root@192.168.22.71:/opt/unetlab/addons/iol/bin/
```
3. Также в директории с IOL-образами должен быть файл с лицензией, который должен называться iourc, содержащий данные о лицензии:  

> [license]  
> eve-ng = <ключ>; 

Скрипт для генерации ключа можно найти [здесь](https://www.ipvanquish.com/2016/09/25/how-to-generate-cisco-iou-licence-on-gns3-vm-with-python-3/).  

После этого при добавлении новой Node можно будет выбрать пункт Cisco IOL, который содержит L2 и L3 образы.  

### Отображать только устройства для которых есть образ

По умолчанию при добавлении нового устройства в лабораторную, появляется список всех устройств (даже если нет установленного образа), что очень неудобно.  
Для исправления ситуации в файле **/opt/unetlab/html/includes/init.php** найти блок кода:  

```
if ( $found == 0 )  {
        //$node_templates[$templ] = $desc.'.missing'  ;
        //$node_templates[$templ] = $desc.TEMPLATE_DISABLED  ;
        unset($node_templates[$templ]) ;
```

Закомментировать строчку **$node_templates[$templ] = $desc.TEMPLATE_DISABLED** и добавить **unset($node_templates[$templ])**, после чего при добавлении устройства будет выбор только тех, чей образ добавлен, как на изображении ниже.  

![Добавление нового устройства](/img/2022-09-13-eve-ng-arch-i3/eve-ng-new-node.png)

## Интеграция с Arch Linux

Прежде всего стоит установить пакет **eve-ng-integration**, который находится в [AUR](https://aur.archlinux.org/packages/eve-ng-integration).  

Если посмотреть внутрь скрипта **/usr/bin/eve-ng-integration**, то видно, что эмулятор терминала выбирается на основе значения переменной окружения **XDG_CURRENT_DESKTOP**.  
Причем по умолчанию в списке графических окружений есть: gnome, kde, xfce4 и т.д.  
В случае, если используется одно из "стандартных" графических окружений, то терминал должен открываться "из коробки".  

### Использование i3wm и терминала alacritty

В случае использования i3wm **echo $XDG_CURRENT_DESKTOP** будет возвращать значение **i3** и по умолчанию будет пытаться запуститься терминал **xterm**, что, скорее всего, не является ожидаемым поведением.  

#### Изменение эмулятора терминала:

1. Для большинства терминалов (к примеру, xfce4-terminal) в файле /usr/bin/eve-ng-integration необходимо найти функцию **terminal_emulator_cmd** и в последнем else закомментировать **return ['xterm', '-e']** и добавить нужный терминал, как в примере ниже.   

```
else:
    #return ['xterm', '-e']
    return ['xfce4-terminal', '-e']
```

2. В случае использования терминала alacritty, описанных выше действий недостаточно, т.к. при выполнении этого скрипта на python, будет передаваться конструкция **alacritty -e 'telnet 192.168.22.71 32769'**, которая работать [не будет](https://github.com/alacritty/alacritty/issues/1266).  
Необходимо передавать терминалу программу и значения не в виде строки (без кавычек), т.е. **alacritty -e telnet 192.168.22.71 32769**.  
Для исправления необходимо в файле **/usr/bin/eve-ng-integration** найти функцию **execute**, закомментировать переменную **command** и добавить такую же, но без *'/n'*, как указано ниже. Теперь терминал, при клике на устройство в eve-ng, должен открываться[^2].  
[^2]: При смене терминала с alacritty на другой, не забыть это исправление в коде 

```
def execute(self, command):
    if isinstance(command, (str)):
        #command = command.split('\n')
        command = command.split()
```

### Использование вкладок в xfce4-terminal и alacritty:

Т.к. для работы с большим количеством устройств удобнее работать с вкладками внутри терминала, а не с каждым окном отдельно. Для этого есть несколько способов:  
1. Использовать терминал, который поддерживает вкладки (xfce4-terminal, konsole)  

> Необходимо учитывать, что по умолчанию, к примеру, xfce4-terminal не будет использовать вкладки.  
> Для их активации необходимо вернуться в предыдущий пункт этой публикации и в строку return ['xfce4-terminal', '-e'] добавить значение '\-\-tab' перед '-e'.  
> Т.е. строка должна выглядеть так: **return ['xfce4-terminal', '\-\-tab', '-e']**  

2. Если используется оконный менеджер (в моем случае i3), то использовать tabbed layout (по умолчанию это mod+w, в конфиге i3 строка bindsym $mod+w layout tabbed).  

Для того, чтобы "отловить" терминалы, созданные из EVE-NG можно воспользоваться способностью alacritty менять class создаваемого окна.  
В файле **/usr/bin/eve-ng-integration** в строке return с терминалом приводим ее к виду **return ['alacritty', '\-\-class', 'eve-ng', '-e']**  
Теперь мы можем это использовать для отправки терминалов, открытых из интерфейса EVE-NG на отдельньный workspace (к примеру, на втором мониторе) и помещать их в tabbed layout.  
Для этого в конфигурационном файле i3 добавим правило:   
```
for_window [class="eve-ng"] move to workspace $ws7 layout tabbed border pixel 0
```
В итоге, средствами оконного менеджера,  получаем альтернативу терминала с владками, которые автоматически открываются на втором мониторе.    
![Вид терминала с "вкладками"](/img/2022-09-13-eve-ng-arch-i3/alacritty-tabs.png)
