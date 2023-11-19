---
created: 2023-10-16T10:37
updated: 2023-11-19T17:10
tags:
  - cisco
  - otus
  - ROS
  - vlan
---
# Маршрутизация между виртуальными локальными сетями. Router-on-a-stick.

## Топология
![](Pasted%20image%2020231019115607.png)
![attachments](https://github.com/2lama2/otus_engineer_pro/blob/master/attachments/Pasted%20image%2020231019115607.png?raw=true)
## Таблица адресации
<table>
<thead>
  <tr>
    <th>Устройство</th>
    <th>Интерфейс</th>
    <th>IP-адрес</th>
    <th>Маска подсети</th>
    <th>Шлюз по умолчанию</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="3">R1</td>
    <td>e0/0.3</td>
    <td>192.168.3.1</td>
    <td>255.255.255.0</td>
    <td rowspan="3">-</td>
  </tr>
  <tr>
    <td>e0/0.4</td>
    <td>192.168.4.1</td>
    <td>255.255.255.0</td>
  </tr>
  <tr>
    <td>e0/0.8</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td>SW1</td>
    <td>VLAN 3</td>
    <td>192.168.3.11</td>
    <td>255.255.255.0</td>
    <td>192.168.3.1</td>
  </tr>
  <tr>
    <td>SW2</td>
    <td>VLAN 3</td>
    <td>192.168.3.12</td>
    <td>255.255.255.0</td>
    <td>192.168.3.1</td>
  </tr>
  <tr>
    <td>VPC1</td>
    <td>NIC</td>
    <td>192.168.3.3</td>
    <td>255.255.255.0</td>
    <td>192.168.3.1</td>
  </tr>
  <tr>
    <td>VPC2</td>
    <td>NIC</td>
    <td>192.168.4.3</td>
    <td>255.255.255.0</td>
    <td>192.168.4.1</td>
  </tr>
</tbody>
</table>

## Таблица VLAN
<table>
<thead>
  <tr>
    <th>VLAN</th>
    <th>Имя</th>
    <th>Назначенный интерфейс</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>3</td>
    <td>MGMT</td>
    <td>SW1: VLAN 3<br>SW2: VLAN 3<br>SW1: e0/2</td>
  </tr>
  <tr>
    <td>4</td>
    <td>OPS</td>
    <td>SW2: e0/1</td>
  </tr>
  <tr>
    <td>7</td>
    <td>PLOT</td>
    <td>SW1: e0/3<br>SW2: e0/2-3</td>
  </tr>
  <tr>
    <td>8</td>
    <td>NATIVE</td>
    <td>-</td>
  </tr>
</tbody>
</table>

## Задачи
1.  Создание сети и настройка основных параметров устройства
2.  Создание сетей VLAN и назначение портов коммутатора
3.  Настройка транка 802.1Q между коммутаторами
4.  Настройка маршрутизации между сетями VLAN
5.  Проверка, что маршрутизация между VLAN работает

---
## 1. Создание сети и настройка основных параметров устройства

Создадим модель сети в **EVE-NG** и выполним базовую настройку маршрутизатора **R1** и коммутаторов **SW1** и **SW2** по следующему шаблону:

![[cisco - базовая конфигурация]]

Назначим IP-адреса виртуальным машинам **`VPC1`** и **`VPC2`** из таблицы.

## 2. Создание сетей VLAN и назначение портов коммутатора

На коммутаторе **SW1** создадим необходимые VLAN'ы и дадим им имена:

```
SW1(config)#vlan 3
SW1(config-vlan)#name MGMT
SW1(config-vlan)#vlan 4
SW1(config-vlan)#name OPS
SW1(config-vlan)#vlan 7
SW1(config-vlan)#name PLOT
SW1(config-vlan)#vlan 8
SW1(config-vlan)#name NATIVE
```
   
Настроим интерфейс управления и шлюз по умолчанию, используя информацию об IP-адресе в таблице адресации:
   
```
SW1(config)#interface vlan 3
SW1(config-if)#ip address 192.168.3.11 255.255.255.0
SW1(config-if)#no shutdown 
SW1(config-if)#exit 
SW1(config)#ip default-gateway 192.168.3.1
```

Назначим все неиспользуемые порты коммутатора (в данном случае **`E0/3`**) VLAN PLOT, переведем их в режим ACCESS и выключим:

```
SW1(config)#interface e0/3
SW1(config-if)#switchport mode access 
SW1(config-if)#switchport access vlan 7
SW1(config-if)#shutdown 
SW1(config-if)#exit
```

Настроим остальные порты коммутатора по таблице VLAN:

```
SW1(config)#int e0/2
SW1(config-if)#switchport mode access 
SW1(config-if)#switchport access vlan 3
SW1(config-if)#exit
SW1(config)#exit 
```

Убедимся в правильности настройки портов:

```
SW1#show vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/0, Et0/1
3    MGMT                             active    Et0/2
4    OPS                              active    
7    PLOT                             active    Et0/3
8    NATIVE                           active    
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 
```
   
Аналогичным образом настроим коммутатор **SW2**:

```
SW2#sh vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/0
3    MGMT                             active    
4    OPS                              active    Et0/1
7    PLOT                             active    Et0/2, Et0/3
8    NATIVE                           active    
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup
```

## 3. Настройка транка 802.1Q между коммутаторами

Проведем настройку магистрального подключения между коммутаторами на стороне коммутатора **SW1**.

Выберем интерфейс и укажем использовать тегирование VLAN'ов по стандарту **IEEE 802.1q**:

```
SW1(config)#int e0/1
SW1(config-if)#switchport trunk encapsulation dot1q 
```

Включим режим магистрали и назначим номер собственной (native) VLAN:

```
SW1(config-if)#switchport mode trunk 
SW1(config-if)#switchport trunk native vlan 8
```

Разрешим проходить по магистрали трафику с тэгом VLAN 3, 4, 7 и 8:

```
SW1(config-if)#switchport trunk allowed vlan 3,4,7,8
```

Проверим правильность сделанных настроек с помощью команды:
   
```
SW1#show interfaces trunk 

Port        Mode             Encapsulation  Status        Native vlan
Et0/1       on               802.1q         trunking      8

Port        Vlans allowed on trunk
Et0/1       3-4,7-8

Port        Vlans allowed and active in management domain
Et0/1       3-4,7-8

Port        Vlans in spanning tree forwarding state and not pruned
Et0/1       3-4,7
```

Аналогичным образом настроим магистральный интерфейс коммутатора **SW2**, обращенный к коммутатору **SW1**:

```
SW2#show interfaces trunk 

Port        Mode             Encapsulation  Status        Native vlan
Et0/0       on               802.1q         trunking      8

Port        Vlans allowed on trunk
Et0/0       3-4,7-8

Port        Vlans allowed and active in management domain
Et0/0       3-4,7-8

Port        Vlans in spanning tree forwarding state and not pruned
Et0/0       3-4,7-8
```

Настроим магистральный интерфейс на коммутаторе **SW1**, обращенный к маршрутизатору **R1**:

```
SW1(config-if)#do sh int trunk

Port        Mode             Encapsulation  Status        Native vlan
Et0/0       on               802.1q         trunking      8
Et0/1       on               802.1q         trunking      8

Port        Vlans allowed on trunk
Et0/0       3-4,7-8
Et0/1       3-4,7-8

Port        Vlans allowed and active in management domain
Et0/0       3-4,7-8
Et0/1       3-4,7-8

Port        Vlans in spanning tree forwarding state and not pruned
Et0/0       3-4,7-8
Et0/1       3-4,7-8
```

## 4. Настройка маршрутизации между сетями VLAN

Настроим маршрутизатор **R1** по таблице адресации:

```
R1#show run | section interface

interface Ethernet0/0
 no ip address
 duplex auto
interface Ethernet0/0.3
 description MGMT
 encapsulation dot1Q 3
 ip address 192.168.3.1 255.255.255.0
interface Ethernet0/0.4
 description OPS
 encapsulation dot1Q 4
 ip address 192.168.4.1 255.255.255.0
interface Ethernet0/0.8
 description NATIVE
 encapsulation dot1Q 8 native
interface Ethernet0/1
 no ip address
 shutdown
 duplex auto
interface Ethernet0/2
 no ip address
 shutdown
 duplex auto
interface Ethernet0/3
 no ip address
 shutdown
 duplex auto
```

Убедимся, что интерфейсы работают:

```
R1# sh ip int br

Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                unassigned      YES unset  up                    up      
Ethernet0/0.3              192.168.3.1     YES manual up                    up      
Ethernet0/0.4              192.168.4.1     YES manual up                    up      
Ethernet0/0.8              unassigned      YES unset  up                    up      
Ethernet0/1                unassigned      YES unset  administratively down down    
Ethernet0/2                unassigned      YES unset  administratively down down    
Ethernet0/3                unassigned      YES unset  administratively down down 
```

## 5. Маршрутизация между VLAN

Проверим, что маршрутизация выполняется верно:

```
VPC1> sh ip                

NAME        : VPC1[1]
IP/MASK     : 192.168.3.3/24
GATEWAY     : 192.168.3.1
DNS         : 
MAC         : 00:50:79:66:68:01
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPC1> ping 192.168.3.1 -c 3

84 bytes from 192.168.3.1 icmp_seq=1 ttl=255 time=0.460 ms
84 bytes from 192.168.3.1 icmp_seq=2 ttl=255 time=0.621 ms
84 bytes from 192.168.3.1 icmp_seq=3 ttl=255 time=0.481 ms

VPC1> ping 192.168.4.3 -c 3   

84 bytes from 192.168.4.3 icmp_seq=1 ttl=63 time=1.024 ms
84 bytes from 192.168.4.3 icmp_seq=2 ttl=63 time=1.091 ms
84 bytes from 192.168.4.3 icmp_seq=3 ttl=63 time=0.825 ms

VPC1> ping 192.168.3.12 -c 3

84 bytes from 192.168.3.12 icmp_seq=1 ttl=255 time=0.177 ms
84 bytes from 192.168.3.12 icmp_seq=2 ttl=255 time=0.413 ms
84 bytes from 192.168.3.12 icmp_seq=3 ttl=255 time=0.348 ms
```

```
VPC2> sh ip

NAME        : VPC2[1]
IP/MASK     : 192.168.4.3/24
GATEWAY     : 192.168.4.1
DNS         : 
MAC         : 00:50:79:66:68:02
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

VPC2> ping 192.168.3.3 -c 3

84 bytes from 192.168.3.3 icmp_seq=1 ttl=63 time=0.963 ms
84 bytes from 192.168.3.3 icmp_seq=2 ttl=63 time=1.149 ms
84 bytes from 192.168.3.3 icmp_seq=3 ttl=63 time=1.353 ms

VPC2> trace 192.168.3.3
trace to 192.168.3.3, 8 hops max, press Ctrl+C to stop
 1   192.168.4.1   0.530 ms  0.302 ms  0.241 ms
 2   *192.168.3.3   0.489 ms (ICMP type:3, code:3, Destination port unreachable)
```

Таким образом, конфигурация проверена на работоспособность. Такая схема имеет специальное название Router-on-a-stick (ROS).
