---
created: 2023-10-16T10:37
updated: 2023-11-19T17:10
---
# Реализация DHCPv4

## Топология
![](2023-10-21_11-36-27.png)
![attachments](https://github.com/2lama2/otus_engineer_pro/blob/master/attachments/2023-10-21_11-36-27.png?raw=true)
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
    <td rowspan="5">R1</td>
    <td>e0/1</td>
    <td>10.0.0.1</td>
    <td>255.255.255.252</td>
    <td rowspan="5">-</td>
  </tr>
  <tr>
    <td>e0/0</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td>e0/0.100</td>
    <td>192.168.1.1</td>
    <td>255.255.255.192</td>
  </tr>
  <tr>
    <td>e0/0.200</td>
    <td>192.168.1.65</td>
    <td>255.255.255.224</td>
  </tr>
  <tr>
    <td>e0/0.1000</td>
    <td>-</td>
    <td>-</td>
  </tr>
  <tr>
    <td rowspan="2">R2</td>
    <td>e0/0</td>
    <td>10.0.0.2</td>
    <td>255.255.255.252</td>
    <td rowspan="2">-</td>
  </tr>
  <tr>
    <td>e0/1</td>
    <td>192.168.1.97</td>
    <td>255.255.255.240</td>
  </tr>
  <tr>
    <td>SW1</td>
    <td>VLAN 200</td>
    <td>192.168.1.66</td>
    <td>255.255.255.224</td>
    <td>192.168.1.65</td>
  </tr>
  <tr>
    <td>SW2</td>
    <td>VLAN 1</td>
    <td>192.168.1.98</td>
    <td>255.255.255.240</td>
    <td>192.168.1.97</td>
  </tr>
  <tr>
    <td>PC-A</td>
    <td>e0/0</td>
    <td>DHCP</td>
    <td>DHCP</td>
    <td>DHCP</td>
  </tr>
  <tr>
    <td>PC-B</td>
    <td>e0/0</td>
    <td>DHCP</td>
    <td>DHCP</td>
    <td>DHCP</td>
  </tr>
</tbody>
</table>

## Таблица VLAN
| **VLAN** | **Имя** | **Назначенный интерфейс** |
|:---------------|:--------------|:-------------|
| 1             | Нет            | S2: e0/1     |
| 100           | CLIENTS        | S1: e0/0     |
| 200           | MGMT           | S1: VLAN 200 |
| 999           | PLOT           | S1: e0/2-3   |
| 1000          | NATIVE         | -            |

## Задачи
1. Создание сети и настройка основных параметров устройства
2. Настройка и проверка двух серверов DHCPv4 на R1
3. Настройка и проверка DHCP-ретрансляции на R2

---

## 1. Создание сети и настройка основных параметров устройства

### Создание схемы адресации

Для того, чтобы организовать поддержку 58 хостов в клиентском сегменте за маршрутизатором R1 (VLAN CLIENTS), 28 хостов в управляющем сегменте (VLAN MGMT), а также 12 хостов в клиентском сегменте за маршрутизатором R2 (VLAN 1), разобьём сеть 192.168.1.0/24 на следующие подсети:

**Подсеть A**: 192.168.1.0 / 26
	- диапазон хостов: 192.168.1.1 - 192.168.1.62
	- маска сети: 255.255.255.192
	- количество хостов: 62
**Подсеть B**: 192.168.1.64 / 27
	- диапазон хостов: 192.168.1.65 - 192.168.1.94
	- маска сети: 255.255.255.224
	- количество хостов: 30
**Подсеть C**: 192.168.1.96 / 28
	- диапазон хостов: 192.168.1.97 - 192.168.1.110
	- маска сети: 255.255.255.240
	- количество хостов: 14

Заполним соответствующими значениями таблицу адресации.

Создадим топологию сети в EVE-NG (где в качестве хостов **PC-A** и **PC-B** будем использовать маршрутизаторы cisco с одним активным интерфейсом **e0/0**) и выполним базовую настройку маршрутизаторов и коммутаторов по шаблону:

![[cisco - базовая конфигурация]]

[cisco - базовая конфигурация](/cisco%20-%20базовая%20конфигурация.md)

### Настроим маршрутизатор **R1**:

```
R1(config-subif)# int e0/0.100
R1(config-subif)# description CLIENTS
R1(config-subif)# encapsulation dot1Q 100
R1(config-subif)# ip address 192.168.1.1 255.255.255.192

R1(config-subif)# int e0/0.200
R1(config-subif)# description MGMT
R1(config-subif)# encapsulation dot1Q 200
R1(config-subif)# ip address 192.168.1.65 255.255.255.224

R1(config-subif)# int e0/0.1000
R1(config-subif)# encapsulation dot1Q 1000 native
R1(config-subif)# description NATIVE

R1(config)# int e0/0
R1(config-if)# no sh

R1(config)# int e0/1
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# no sh

R1(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.2

R1# sh ip int br

Interface                  IP-Address      OK? Method Status               Protocol
Ethernet0/0                unassigned      YES unset  up                    up  
Ethernet0/0.100            192.168.1.1     YES manual up                    up  
Ethernet0/0.200            192.168.1.65    YES manual up                    up  
Ethernet0/0.1000           unassigned      YES unset  up                    up  
Ethernet0/1                10.0.0.1        YES manual up                    up  
Ethernet0/2                unassigned      YES unset  administratively down down
Ethernet0/3                unassigned      YES unset  administratively down down
```

### Настроим маршрутизатор **R2**:

```
R2(config)# int e0/1
R2(config-if)# ip address 192.168.1.97 255.255.255.240
R2(config-if)# no sh

R2(config-if)# int e0/0
R2(config-if)# ip address 10.0.0.2 255.255.255.252
R2(config-if)# no sh

R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1

R2# sh ip int brief

Interface                  IP-Address      OK? Method Status               Protocol
Ethernet0/0                10.0.0.2        YES manual up                    up  
Ethernet0/1                192.168.1.97    YES manual up                    up  
Ethernet0/2                unassigned      YES unset  administratively down down
Ethernet0/3                unassigned      YES unset  administratively down down
```

### Выполним настройки коммутаторов

Ииспользуя вышеприведенную информацию об адресации и конфигурации VLAN, настроим коммутаторы **SW1**:

```
SW1(config)#vlan 100
SW1(config-vlan)#name CLIENTS
SW1(config-vlan)#vlan 200
SW1(config-vlan)#name MGMT
SW1(config-vlan)#vlan 999
SW1(config-vlan)#name PLOT
SW1(config-vlan)#vlan 1000 
SW1(config-vlan)#name NATIVE

SW1(config-vlan)#int vlan 200
SW1(config-if)#ip address 192.168.1.66 255.255.255.224
SW1(config-if)#no sh
SW1(config-if)#exi

SW1(config)#ip default-gateway 192.168.1.65

SW1(config-if)#int range e0/2-3
SW1(config-if-range)#switchport mode access
SW1(config-if-range)#sw access vlan 999
SW1(config-if-range)#sh

SW1(config-if-range)#int e0/0
SW1(config-if)#sw mode access 
SW1(config-if)#sw access vlan 100

SW1(config-if)#int e0/1
SW1(config-if)#sw trunk enc dot
SW1(config-if)#sw mode trunk
SW1(config-if)#sw trunk native vlan 1000
SW1(config-if)#sw trunk allowed lvan 100,200,1000
```

**SW2:**

```
SW2(config-vlan)#vlan 999
SW2(config-vlan)#name PLOT

SW2(config-vlan)#int vlan 1
SW2(config-if)#ip address 192.168.1.98 255.255.255.240
SW2(config-if)#no sh
SW2(config-if)#exi

SW2(config)#ip default-gateway 192.168.1.97

SW2(config-if)#int range e0/2-3
SW2(config-if-range)#switchport mode access
SW2(config-if-range)#sw access vlan 999
SW2(config-if-range)#sh
```

Проверим сделанные настройки:

```
SW1(config-if)#do sh int tr         

Port        Mode             Encapsulation  Status        Native vlan
Et0/1       on               802.1q         trunking      1000

Port        Vlans allowed on trunk
Et0/1       100,200,1000

Port        Vlans allowed and active in management domain
Et0/1       100,200,1000

Port        Vlans in spanning tree forwarding state and not pruned
Et0/1       100,200,1000

SW1#sh vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
100  CLIENTS                          active    Et0/0
200  MGMT                             active    
999  PLOT                             active    Et0/2, Et0/3
1000 NATIVE                           active    
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 

SW1#sh ip int br
Interface              IP-Address      OK? Method Status                Protocol
Ethernet0/0            unassigned      YES unset  up                    up      
Ethernet0/1            unassigned      YES unset  up                    up      
Ethernet0/2            unassigned      YES unset  administratively down down    
Ethernet0/3            unassigned      YES unset  administratively down down    
Vlan200                192.168.1.66    YES manual up                    up      
```

```
SW2#sh vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/0, Et0/1
999  PLOT                             active    Et0/2, Et0/3
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 

SW2#sh ip int brief 
Interface              IP-Address      OK? Method Status                Protocol
Ethernet0/0            unassigned      YES unset  up                    up      
Ethernet0/1            unassigned      YES unset  up                    up      
Ethernet0/2            unassigned      YES unset  administratively down down    
Ethernet0/3            unassigned      YES unset  administratively down down    
Vlan1                  192.168.1.98    YES manual up                    up      
```

### 2. Настройка и проверка двух серверов DHCPv4 на R1

Настроим два DHCP-пула для подсети A и C:

Создадим пул со следующими параметрами:
	- имя пула: **R1_Client_LAN**
	- сеть: **192.168.1.0 / 26**
	- шлюз: **192.168.1.1**
	- имя домена: **CCNA-lab.com**
	- время аренды: **2дн 14час 30мин**

Создадим второй пул с такими параметрами:
	- имя пула: **R2_Client_LAN**
	- сеть: **192.168.1.96 / 28**
	- шлюз: **192.168.1.97**
	- имя домена: **CCNA-lab.com**
	- время аренды: **2дн 14час 30мин**

Исключим из автоматической выдачи первые 5 адресов подсети A и первые 6 адресов подсети C.

```
R1(config)#ip dhcp excluded-address 192.168.1.1 192.168.1.5
R1(config)#ip dhcp excluded-address 192.168.1.97 192.168.1.102

R1(config)#ip dhcp pool R1_Client_LAN
R1(dhcp-config)#network 192.168.1.0 255.255.255.192
R1(dhcp-config)#domain-name CCNA-lab.com
R1(dhcp-config)#default-router 192.168.1.1
R1(dhcp-config)#lease 2 12 30
R1(dhcp-config)#exit

R1(config)#ip dhcp pool R2_Client_LAN
R1(dhcp-config)#network 192.168.1.96 255.255.255.240
R1(dhcp-config)#domain-name CCNA-lab.com
R1(dhcp-config)#default-router 192.168.1.97
R1(dhcp-config)#lease 2 12 30
```

На хосте **PC-A** настроим сетевой интерфейс **e0/0**  для получения настроек протокола IPv4 в автоматическом режиме:

```
PC-A(config)#interface e0/0
PC-A(config-if)#ip address dhcp
```

На стороне клиента **PC-A** проверим автоконфигурацию сетевого интерфейса и доступность шлюза по умолчанию:

```
PC-A#sh dhcp lease 

Temp IP addr: 192.168.1.6  for peer on Interface: Ethernet0/0
Temp  sub net mask: 255.255.255.192
   DHCP Lease server: 192.168.1.1, state: 5 Bound
   DHCP transaction id: AD7
   Lease: 217800 secs,  Renewal: 108900 secs,  Rebind: 190575 secs
Temp default-gateway addr: 192.168.1.1
   Next timer fires after: 1d05h
   Retry count: 0   Client-ID: cisco-aabb.cc00.7000-Et0/0
   Client-ID hex dump: 636973636F2D616162622E636330302E
                       373030302D4574302F30
   Hostname: PC-A

PC-A#ping 192.168.1.1 

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

Ожидаемо ip-адрес получен динамически из пула для подсети A. Шлюз по умолчанию также доступен.

### 3. Настройка и проверка DHCP-ретрансляции на R2

Настроим ретрансляцию запросов DHCP из подсети C на DHCP-сервер с адресом 10.0.0.1 с помощью команды **ip helper-address**:

```
R2(config)#int e0/1
R2(config-if)#ip helper-address 10.0.0.1
```
  
На стороне клиента **PC-B** проверим автоконфигурацию сетевого интерфейса и доступность шлюза по умолчанию:

```
PC-B(config)#interface e0/0
PC-B(config-if)#ip address dhcp

PC-B#sh dhcp lease 

Temp IP addr: 192.168.1.103  for peer on Interface: Ethernet0/0
Temp  sub net mask: 255.255.255.240
   DHCP Lease server: 10.0.0.1, state: 5 Bound
   DHCP transaction id: AF0
   Lease: 217800 secs,  Renewal: 108900 secs,  Rebind: 190575 secs
Temp default-gateway addr: 192.168.1.97
   Next timer fires after: 1d05h
   Retry count: 0   Client-ID: cisco-aabb.cc00.8000-Et0/0
   Client-ID hex dump: 636973636F2D616162622E636330302E
                       383030302D4574302F30
   Hostname: PC-B

PC-B#ping 192.168.1.97

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.97, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
  
Здесь также интерфейс получил адрес автоматически из своего пула подсети C. Шлюз доступен.

Посмотрим статистику DHCP-сервера:

```
R1#sh ip dhcp server statistics

Memory usage         42133
Address pools        2
Database agents      0
Automatic bindings   2
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         2
DHCPREQUEST          2
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            2
DHCPACK              2
DHCPNAK              0
```
  
и выданные в аренду IP-адреса:

```
R1#sh ip dhcp binding

Bindings from all pools not associated with VRF:
IP address          Client-ID/	 	    Lease expiration        Type
		    Hardware address/
		    User name
192.168.1.6         0063.6973.636f.2d61.    Oct 23 2023 09:06 PM    Automatic
                    6162.622e.6363.3030.
                    2e37.3030.302d.4574.
                    302f.30
192.168.1.103       0063.6973.636f.2d61.    Oct 23 2023 09:06 PM    Automatic
                    6162.622e.6363.3030.
                    2e38.3030.302d.4574.
                    302f.30
```
