---
created: 2023-10-16T10:37
updated: 2023-11-19T17:10
---
# Настройка DHCPv6

## Топология
![](2023-10-21_11-36-27.png)
![attachments](https://github.com/2lama2/otus_engineer_pro/blob/master/attachments/2023-10-21_11-36-27.png?raw=true)
## Таблица адесации

<table>
<thead>
  <tr>
    <th>Устройство</th>
    <th>Интерфейс</th>
    <th>IPv6-адрес</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="4">R1</td>
    <td rowspan="2">e0/1</td>
    <td>2001:db8:acad:2::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td rowspan="2">e0/0</td>
    <td>2001:db8:acad:1::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td rowspan="4">R2</td>
    <td rowspan="2">e0/0</td>
    <td>2001:db8:acad:2::2/64</td>
  </tr>
  <tr>
    <td>fe80::2</td>
  </tr>
  <tr>
    <td rowspan="2">e0/1</td>
    <td>2001:db8:acad:3::1/64</td>
  </tr>
  <tr>
    <td>fe80::1</td>
  </tr>
  <tr>
    <td>PC-A</td>
    <td>e0/0</td>
    <td>DHCP</td>
  </tr>
  <tr>
    <td>PC-B</td>
    <td>e0/0</td>
    <td>DHCP</td>
  </tr>
</tbody>
</table>

## Задачи

1. Создание сети и настройка основных параметров устройства
2. Проверка назначения адреса SLAAC от R1
3. Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1
4. Настройка и проверка DHCPv6-сервера с отслеживанием состояния на R1
5. Настройка и проверка DHCPv6 Relay на R2

---

## 1. Создание сети и настройка основных параметров устройства

Продолжим работу со [схемой](05%20-%20Протокол%20DHCPv4.md), созданной в задании по настройке DHCPv4.

### Настроим маршрутизатор **R1**:

```
R1(config)#ipv6 unicast-routing

R1(config)#int e0/1
R1(config-if)#ipv6 address 2001:db8:acad:2::1/64
R1(config-if)#ipv6 address fe80::1 link-local

R1(config)#int e0/0.100
R1(config-if)#ipv6 address 2001:db8:acad:1::1/64
R1(config-if)#ipv6 address fe80::1 link-local
```

### Настроим маршрутизатор **R2**:

```
R2(config)#ipv6 unicast-routing

R2(config)#int e0/1
R2(config-if)#ipv6 address 2001:db8:acad:3::1/64
R2(config-if)#ipv6 address fe80::1 link-local

R2(config)#int e0/0
R2(config-if)#ipv6 address 2001:db8:acad:2::2/64
R2(config-if)#ipv6 address fe80::1 link-local
```

На обоих маршрутизаторах установим маршрутом по умолчанию соответствующий адрес интерфейса противоположного маршрутизатора:

```
R1(config)# ipv6 route ::/0 e0/1 fe80::2

R2(config)# ipv6 route ::/0 e0/0 fe80::1
```
 
Командой **PING** проверим, что маршрутизаторы видят друг друга:

```
R2#ping ipv6 fe80::1%Ethernet0/0

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FE80::1, timeout is 2 seconds:
Packet sent with a source address of FE80::2%Ethernet0/0
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

## 2. Проверка назначения адреса SLAAC от R1

На хосте **PC-A** настроим сетевой интерфейс **e0/0** для использования протокола IPv6 в режиме автоконфигурации (SLAAC):

```
PC-A(config)#interface e0/0
PC-A(config-if)#ipv6 enable
PC-A(config-if)#ipv6 address autoconfig

PC-A#sh ipv6 int br

Ethernet0/0            [up/up]
    FE80::A8BB:CCFF:FE00:7000
    2001:DB8:ACAD:1:A8BB:CCFF:FE00:7000
Ethernet0/1            [administratively down/down]
    unassigned
Ethernet0/2            [administratively down/down]
    unassigned
Ethernet0/3            [administratively down/down]
    unassigned

PC-A#sh ipv6 route

IPv6 Routing Table - default - 4 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, HA - Home Agent, MR - Mobile Router, R - RIP
       H - NHRP, I1 - ISIS L1, I2 - ISIS L2, IA - ISIS interarea
       IS - ISIS summary, D - EIGRP, EX - EIGRP external, NM - NEMO
       ND - ND Default, NDp - ND Prefix, DCE - Destination, NDr - Redirect
       RL - RPL, O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1
       OE2 - OSPF ext 2, ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2
       la - LISP alt, lr - LISP site-registrations, ld - LISP dyn-eid
       lA - LISP away, a - Application
ND  ::/0 [2/0]
     via FE80::1, Ethernet0/0
NDp 2001:DB8:ACAD:1::/64 [2/0]
     via Ethernet0/0, directly connected
L   2001:DB8:ACAD:1:A8BB:CCFF:FE00:7000/128 [0/0]
     via Ethernet0/0, receive
L   FF00::/8 [0/0]
     via Null0, receive
```
  
Вывод команды показывает, что адрес хоста сформирован префиксом сети **2001:db8:acad:1::/64**, который установлен на маршрутизаторе **R1** и идентификатор интерфейса сформирован методом EUI-64 - **A8BB:CCFF:FE00:7000**. А также шлюзом по умолчанию установлен link-local адрес маршрутизатора - **fe80::1**.

## 3. Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1

Настроим DHCPv6-сервер на маршрутизаторе R1:

- создадим пул адресов:
```
R1(config)#ipv6 dhcp pool R1-STATELESS
```

- зададим имя DNS-сервера и доменный суффикс:

```
R1(config-dhcpv6)#dns-server 2001:db8:acad::254
R1(config-dhcpv6)#domain-name STATELESS.COM
```
 
- на интерфейсе, смотрящем в клиентский сегмент, настроим параметры сообщения **RA** протокола **ICMPv6** так, чтобы клиенты для автоконфигурации IPv6 обращались на DHCP-сервер за дополнительными настройками (устанавим флаг **O**):

```
R1(config-dhcpv6)#int e0/0.100
R1(config-if)#ipv6 nd other-config-flag
```

- привяжем интерфейс к созданному пулу адресов
```
R1(config-if)#ipv6 dhcp server R1-STATELESS
```
 
Проверим вновь полученные параметры автоконфигурации на клиенте **PC-A** после изменения режима получения адреса:

```
PC-A(config)#int e0/0
PC-A(config-if)#ipv6 address dhcp

PC-A#sh ipv6 dhcp interface

Ethernet0/0 is in client mode
  Prefix State is IDLE (0)
  Information refresh timer expires in 23:59:15
  Address State is SOLICIT (8)
  Retransmission timer expires in 00:01:27
  List of known servers:
    Reachable via address: FE80::1
    DUID: 00030001AABBCC005000
    Preference: 0
    Configuration parameters:
      DNS server: 2001:DB8:ACAD::254
      Domain name: STATELESS.com
      Information refresh time: 0
  Prefix Rapid-Commit: disabled
  Address Rapid-Commit: disabled

PC-A#sh ipv6 int br

Ethernet0/0            [up/up]
    FE80::A8BB:CCFF:FE00:7000
    2001:DB8:ACAD:1:A8BB:CCFF:FE00:7000
Ethernet0/1            [administratively down/down]
    unassigned
Ethernet0/2            [administratively down/down]
    unassigned
Ethernet0/3            [administratively down/down]
    unassigned
```

Вывод команды показывает, что интерфейс получил параметры dns-сервера и имени домена, а также сгенерировал глобальный адрес с полученным от маршрутизатора префиксом.

Шлюзом по умолчанию назначен link-local адрес интерфейса маршрутизатора, подключенного к клиентской сети:

```
PC-A#sh ipv6 route

IPv6 Routing Table - default - 4 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, HA - Home Agent, MR - Mobile Router, R - RIP
       H - NHRP, I1 - ISIS L1, I2 - ISIS L2, IA - ISIS interarea
       IS - ISIS summary, D - EIGRP, EX - EIGRP external, NM - NEMO
       ND - ND Default, NDp - ND Prefix, DCE - Destination, NDr - Redirect
       RL - RPL, O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1
       OE2 - OSPF ext 2, ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2
       la - LISP alt, lr - LISP site-registrations, ld - LISP dyn-eid
       lA - LISP away, a - Application
ND  ::/0 [2/0]
     via FE80::1, Ethernet0/0
NDp 2001:DB8:ACAD:1::/64 [2/0]
     via Ethernet0/0, directly connected
L   2001:DB8:ACAD:1:A8BB:CCFF:FE00:7000/128 [0/0]
     via Ethernet0/0, receive
L   FF00::/8 [0/0]
     via Null0, receive
```

Проверим доступность второй клиентской сети командой **PING** до IPv6-адреса маршрутизатора **R2**:

```
PC-A#ping 2001:db8:acad:3::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:3::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/5/21 ms
```

## 4. Настройка и проверка DHCPv6-сервера с отслеживанием состояния на R1

Настроим маршрутизатор **R1** для ответа на запросы DHCPv6 из локальной сети за маршрутизатором **R2**.

Создадим пул для второй клиентской сети:

```
R1(config)#ipv6 dhcp pool R2-STATEFUL
```
 
Назначим префикс адреса:

```
R1(config-dhcpv6)#address prefix 2001:db8:acad:3:aaa::/80
```
 
Зададим имя DNS-сервера и доменный суффикс:

```
R1(config-dhcpv6)#dns-server 2001:db8:acad::254
R1(config-dhcpv6)#domain-name STATEFUL.com
```
 
На интерфейсе, который подключен в сеть с маршрутизатором **R2**, назначим DHCPv6-сервер с пулом **R2-STATEFUL**:

```
R1(config-dhcpv6)#int e0/1
R1(config-if)#ipv6 dhcp server R2-STATEFUL
```

## 5. Настройка и проверка DHCPv6 Relay на R2

Настроим на хосте **PC-B** сетевой интерфейс для использования протокола IPv6 в режиме **SLAAC**:

```
PC-B(config)#int e0/0
PC-B(config-if)#ipv6 enable 
PC-B(config-if)#ipv6 address autoconfig
```
 
Проверим полученные настройки:

```
PC-B#sh ipv6 int br      

Ethernet0/0            [up/up]
    FE80::A8BB:CCFF:FE00:8000
    2001:DB8:ACAD:3:A8BB:CCFF:FE00:8000
Ethernet0/1            [administratively down/down]
    unassigned
Ethernet0/2            [administratively down/down]
    unassigned
Ethernet0/3            [administratively down/down]
    unassigned

PC-B#sh ipv6 route       

IPv6 Routing Table - default - 4 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, HA - Home Agent, MR - Mobile Router, R - RIP
       H - NHRP, I1 - ISIS L1, I2 - ISIS L2, IA - ISIS interarea
       IS - ISIS summary, D - EIGRP, EX - EIGRP external, NM - NEMO
       ND - ND Default, NDp - ND Prefix, DCE - Destination, NDr - Redirect
       RL - RPL, O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1
       OE2 - OSPF ext 2, ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2
       la - LISP alt, lr - LISP site-registrations, ld - LISP dyn-eid
       lA - LISP away, a - Application
ND  ::/0 [2/0]
     via FE80::2, Ethernet0/0
NDp 2001:DB8:ACAD:3::/64 [2/0]
     via Ethernet0/0, directly connected
L   2001:DB8:ACAD:3:A8BB:CCFF:FE00:8000/128 [0/0]
     via Ethernet0/0, receive
L   FF00::/8 [0/0]
     via Null0, receive
```

Вывод команд показывает, что адрес хоста сформирован префиксом сети **2001:db8:acad:3::/64**, установленным на маршрутизаторе **R2** на интерфейсе **e0/1** и идентификатор интерфейса сформирован методом EUI-64 - **A8BB:CCFF:FE00:8000**, то есть был использован механизм SLAAC. А также шлюзом по умолчанию установлен link-local адрес маршрутизатора **R2** - **fe80::2**.

Настроим маршрутизатор **R2** в качестве агента DHCPv6-ретрансляции для второй клиентской сети:

```
R2(config)#int e0/1
R2(config-if)#ipv6 nd managed-config-flag
R2(config-if)#ipv6 dhcp relay destination fe80::1 Ethernet 0/0
```

Командой **ipv6 nd managed-config-flag** устанавливаем флаг **M** в сообщениях **RA** протокола **ICMPv6**, чтобы клиенты для автоконфигурации IPv6 обращались на DHCPv6-сервер за всеми настройками, кроме шлюза по умолчанию, который по-прежнему берется клиентом из **RA**-сообщения маршрутизатора.

Командой **ipv6 dhcp relay destination fe80::1 Ethernet 0/0** включаем relay-агента на интерфейсе маршрутизатора в локальной сети, который будет пересылать запросы к DHCPv6-серверу из сети клиентов к указанному адресу DHCPv6-сервера (в данном случае - к маршрутизатору **R1**, используя его link-local-адрес, через свой интерфейс **e0/0**).

На хосте **PC-B** настроим сетевой интерфейс для получения параметров протокола IPv6 с DHCPv6-сервера:

```
PC-B(config)#int e0/0
PC-B(config-if)#ipv6 enable
PC-B(config-if)#no ipv6 address autoconfig
PC-B(config-if)#ipv6 address dhcp

PC-B(config-if)#sh
PC-B(config-if)#no sh
```
 
Проверим настройки, полученные от DHCPv6-сервера:

```
PC-B#sh ipv6 dhcp int

Ethernet0/0 is in client mode
  Prefix State is IDLE
  Address State is OPEN
  Renew for address will be sent in 11:57:04
  List of known servers:
    Reachable via address: FE80::2
    DUID: 00030001AABBCC005000
    Preference: 0
    Configuration parameters:
      IA NA: IA ID 0x00030001, T1 43200, T2 69120
        Address: 2001:DB8:ACAD:3:AAA:4DA8:8C33:C41E/128
                preferred lifetime 86400, valid lifetime 172800
                expires at Oct 23 2023 01:16 AM (172624 seconds)
      DNS server: 2001:DB8:ACAD::254
      Domain name: STATEFUL.com
      Information refresh time: 0
  Prefix Rapid-Commit: disabled
  Address Rapid-Commit: disabled

PC-B(config-if)#do sh ipv6 int e0/0

Ethernet0/0 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::A8BB:CCFF:FE00:8000 
  No Virtual link-local address(es):
  Global unicast address(es):
    2001:DB8:ACAD:3:AAA:4DA8:8C33:C41E, subnet is 2001:DB8:ACAD:3:AAA:4DA8:8C33:C41E/128 
  Joined group address(es):
    FF02::1
    FF02::1:FF00:8000
    FF02::1:FF33:C41E
  MTU is 1500 bytes
  ICMP error messages limited to one every 100 milliseconds
  ICMP redirects are enabled
  ICMP unreachables are sent
  ND DAD is enabled, number of DAD attempts: 1
  ND reachable time is 30000 milliseconds (using 30000)
  ND NS retransmit interval is 1000 milliseconds
  Default router is FE80::2 on Ethernet0/0
```
 
После того, как мы настроили relay-агента, наш клиент смог получить все параметры (кроме шлюза по умолчанию) от DHCPv6-сервера на маршрутизаторе **R1**. А именно: адрес получен с нужным префиксом, настроенным в пуле адресов для данной сети, также получены настройки DNS-сервера и суффикса доменного имени.

Информацию о назначенных адресах клиентам можно посмотреть на DHCP-сервере с помощью команды:

```
R1#sh ipv6 dhcp binding

Client: FE80::A8BB:CCFF:FE00:8000 
  DUID: 00030001AABBCC008000
  Username : unassigned
  VRF : default
  IA NA: IA ID 0x00030001, T1 43200, T2 69120
    Address: 2001:DB8:ACAD:3:AAA:4DA8:8C33:C41E
            preferred lifetime 86400, valid lifetime 172800
            expires at Oct 23 2023 01:16 AM (172214 seconds)
```

Проверим доступность первой клиентской сети из второй, используя команду **PING** до адреса интерфейса маршрутизатора **R1** c клиента **PC-B**:

```
PC-B#ping ipv6 2001:db8:acad:1::1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:1::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/4/17 ms
```
