```
HOSTNAME# clock set HH:MM:SS DAY MONTH YEAR
HOSTNAME# configure terminal
HOSTNAME(config)# hostname HOSTNAME

HOSTNAME(config)# enable secret PASSWORD
HOSTNAME(config)# service password-encryption

HOSTNAME(config)# banner motd #Warning! Ye've been warned!#

HOSTNAME(config)# no ip domain-lookup

HOSTNAME(config)# line console 0
HOSTNAME(config-line)# password PASSWORD
HOSTNAME(config-line)# login

HOSTNAME(config-line)# logging synchronous

HOSTNAME(config)# line vty 0 4
HOSTNAME(config-line)# password PASSWORD
HOSTNAME(config-line)# login
HOSTNAME(config-line)# transport input telnet

HOSTNAME# copy running-config startup-config
```
