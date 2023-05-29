# Сетевые пакеты. VLAN'ы. LACP.

Задание:

1. В Office1 в тестовой подсети появляется сервера с доп интерфесами и адресами
в internal сети testLAN:
 
- testClient1 - 10.10.10.254

- testClient2 - 10.10.10.254

- testServer1- 10.10.10.1 

- testServer2- 10.10.10.1

2. Равести вланами:

- testClient1 <-> testServer1

- testClient2 <-> testServer2

3. Между `centralRouter` и `inetRouter` "пробросить" 2 линка (общая inernal сеть) и объединить их в `bond`, проверить работу c отключением интерфейсов.

Цель:

Создать домашнюю сетевую лабораторию. И научиться настраивать VLAN и LACP.

## Развертывание стенда для демострации настройки DNS.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и семи виртуальных машин `centos/7`.

В исходном Vagranfile я изменил колличество ядер и объем оперативной памяти до 256 Mb каждой ВМ, так как моея хостовая машина ограничена по ресурсам и такрой стенд не поднимет.

По этим же соображениям ОС ВМ `testClient2` и `testServer2` также была заменена на `centos/7`, так как `ubuntu/focal64` на объявленных ресурсах для ВМ не поднимется.

Vagranfile описывает создание 7 виртуальных машин на CentOS 7, каждой машине будет выделено по 256 МБ ОЗУ. В файле есть модуль, который отвечает за настройку ВМ с помощью Ansible.

Машины с именами: `inetRouter`, `centralRouter` и `office1Router` выполняют роль роутеров.

Машины с именами: `testClient1`, `testServer1`, `testClient2` и `testServer2` выполняют роль клиентов.

Схема коммутации:

![Снимок экрана от 2023-05-26 17-33-09](https://github.com/skyfly535/23WH_VLAN/assets/114483769/4b16cc90-ae49-4d3e-9858-c165d49f5a75)

Разворачиваем инфраструктуру в Vagrant исключительно через Ansible.

Все коментарии по каждому блоку указаны в тексте Playbook - `vlan.yml`.

Файл `hosts` необходимо поместить в каталог с Playbook `vlan.yml` и Vagranfile.

Выполняем установку стенда:

```
vagrant up
```

## Результат работы

### Настройка VLAN на хостах

Проверим настройку сетевых интерфейсов на `testClient1` и пропингуем `testServer1`:
```
[root@testServer1 vagrant]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 81655sec preferred_lft 81655sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:c7:35:f0 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::9cbd:f427:16ee:8d7b/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
5: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:2d:d3:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.22/24 brd 192.168.56.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe2d:d3cf/64 scope link 
       valid_lft forever preferred_lft forever
6: eth1.1@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:c7:35:f0 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.13/24 brd 10.10.10.255 scope global noprefixroute eth1.1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fec7:35f0/64 scope link 
       valid_lft forever preferred_lft forever
[root@testServer1 vagrant]# ping 10.10.10.15
PING 10.10.10.15 (10.10.10.15) 56(84) bytes of data.
64 bytes from 10.10.10.15: icmp_seq=1 ttl=64 time=1.25 ms
64 bytes from 10.10.10.15: icmp_seq=2 ttl=64 time=1.37 ms
64 bytes from 10.10.10.15: icmp_seq=3 ttl=64 time=1.02 ms
64 bytes from 10.10.10.15: icmp_seq=4 ttl=64 time=1.34 ms
^C
--- 10.10.10.15 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3008ms
rtt min/avg/max/mdev = 1.024/1.251/1.379/0.140 ms
```

Повторяем все действия на `testServer1`:

```
[root@testClient1 vagrant]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 82164sec preferred_lft 82164sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ee:13:80 brd ff:ff:ff:ff:ff:ff
5: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:06:5b:87 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.21/24 brd 192.168.56.255 scope global noprefixroute eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe06:5b87/64 scope link 
       valid_lft forever preferred_lft forever
6: eth1.1@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 08:00:27:ee:13:80 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.15/24 brd 10.10.10.255 scope global noprefixroute eth1.1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feee:1380/64 scope link 
       valid_lft forever preferred_lft forever
[root@testClient1 vagrant]# ping 10.10.10.13
PING 10.10.10.13 (10.10.10.13) 56(84) bytes of data.
64 bytes from 10.10.10.13: icmp_seq=1 ttl=64 time=1.19 ms
64 bytes from 10.10.10.13: icmp_seq=2 ttl=64 time=1.24 ms
64 bytes from 10.10.10.13: icmp_seq=3 ttl=64 time=1.31 ms
^C
--- 10.10.10.13 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 1.193/1.248/1.311/0.063 ms
```
Пробуем пропинговать `testServer2`:

```
[root@testServer1 vagrant]# ping 10.10.10.1 
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
^C
--- 10.10.10.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2007ms
```

```
[root@testClient1 vagrant]# ping 10.10.10.1 
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
^C
--- 10.10.10.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1003ms
```
Наблюдаем разделение клиентов по vlan-ам.
### Настройка LACP между хостами inetRouter и centralRouter:

Для проверки работы bond-интерфейса на хосте `inetRouter` (192.168.255.1) запустим ping до `centralRouter` (192.168.255.2), после чего на `centralRouter` поочередно отключаем сетевые интерфейсы задействованные при организации bond-интерфейса:

![Снимок экрана от 2023-05-26 17-31-54](https://github.com/skyfly535/23WH_VLAN/assets/114483769/0db99149-f268-437b-a751-eb80b36b4722)

на снимке терминалов видно, что после отключения второго интерфейса пинг пропадает (9-14 секунды), а после включения одного из интерфейсо пакеты продолжают доходить до адресата.
