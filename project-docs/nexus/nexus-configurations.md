# Nexus Consulting Group: Device Configurations

> Companion file to `nexus-documentation.md`. Contains the running configuration commands for every device type referenced in that document. Router0 and the DMZ server configs are not included; they weren't supplied as part of this configuration set.

---

## Organisation profile (for reference)

Nexus Consulting Group is a Kampala-based professional services firm specialising in management consulting, financial advisory, and HR solutions for mid-size organisations across East Africa. The firm operates from a single office premises in Nakasero, Kampala, with a staff of 70 across four departments.

---

## CORE-SWITCH1 (C-SW1)

```
en
conf t
hostname C-SW1
ip routing

vtp domain nexus
vtp mode server

username admin privilege 15 secret admin123
username support privilege 1 secret support123
ip domain-name nexus.local
crypto key gen rsa general-keys mod 2048

line console 0
login local
exit
enable algorithm-type scrypt secret cisco-nexus-123

line vty 0 15
password cisco-nexus-123 
login
transport input ssh
login local
exit
service password-encryption

vlan 10
name IT
vlan 20
name HR
vlan 30
name SERVERS
vlan 40 
name SALES
vlan 50
name FINANCE

int range g1/0/6-7
switchport
channel-group 1 mode active
channel-protocol lacp
ip arp inspection trust
ip dhcp snooping trust

int port-channel 1
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60
ip arp inspection trust
ip dhcp snooping trust

int vl 10
no shut
ip add 192.168.1.2 255.255.255.0
standby 10 priority 110
standby 10 ip 192.168.1.1 
standby 10 preempt
ip helper-address 192.168.3.4

int vl 20
no shut
ip add 192.168.2.2 255.255.255.0
standby 20 priority 110
standby 20 ip 192.168.2.1 
standby 20 preempt
ip helper-address 192.168.3.4

int vl 30
no shut
ip add 192.168.3.2 255.255.255.0
standby 30 priority 110
standby 30 ip 192.168.3.1 
standby 30 preempt

int vl 40
no shut
ip add 192.168.4.2 255.255.255.0
standby 40 priority 100
standby 40 ip 192.168.4.1
ip helper-address 192.168.3.4

int vl 50
no shut
ip add 192.168.5.2 255.255.255.0
standby 50 priority 100
standby 50 ip 192.168.5.1 
ip helper-address 192.168.3.4

int g1/0/1
no switchport
no shut 
ip add 10.0.0.1 255.255.255.252
ex

router ospf 15
router-id 1.2.1.2
network 192.168.0.0 0.0.7.255 area 0
network 10.0.0.0 0.0.0.3 area 0
ex

int range g1/0/2-5
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60
ex

spanning-tree vlan 10,20,30 root primary 
spanning-tree vlan 40,50,60 root secondary
spanning-tree mode rapid-pvst

ip dhcp snooping 
ip dhcp snooping vlan 10,20,30,40,50,60
no ip dhcp snooping information option 
ip arp inspection vlan 10,20,30,40,50,60
ip arp inspection validate dst-mac ip src-mac

ip route 0.0.0.0 0.0.0.0 10.0.0.2

access-list 100 permit icmp 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 100 deny icmp 192.168.0.0 0.0.7.255 192.168.3.0 0.0.0.255
access-list 100 permit ip any any

access-list 120 permit icmp 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 120 deny icmp 192.168.3.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 120 permit ip any any
int vlan 30
ip access-group 100 out
ip access-group 120 in
ex

access-list 101 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 permit ip 192.168.3.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 deny ip 192.168.0.0 0.0.7.255 192.168.2.0 0.0.0.255
access-list 101 permit ip any any

access-list 110 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 110 permit ip 192.168.2.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 110 deny ip 192.168.2.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 110 permit ip any any
int vlan 20
ip access-group 101 out
ip access-group 110 in
ex

access-list 102 permit ip 192.168.1.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 102 permit ip 192.168.3.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 102 deny ip 192.168.0.0 0.0.7.255 192.168.4.0 0.0.0.255
access-list 102 permit ip any any

access-list 111 permit ip 192.168.4.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 111 permit ip 192.168.4.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 111 deny ip 192.168.4.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 111 permit ip any any
int vlan 40
ip access-group 102 out
ip access-group 111 in
ex

access-list 103 permit ip 192.168.1.0 0.0.0.255 192.168.5.0 0.0.0.255
access-list 103 permit ip 192.168.3.0 0.0.0.255 192.168.5.0 0.0.0.255
access-list 103 deny ip 192.168.0.0 0.0.7.255 192.168.5.0 0.0.0.255
access-list 103 permit ip any any

access-list 112 permit ip 192.168.5.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 112 permit ip 192.168.5.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 112 deny ip 192.168.5.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 112 permit ip any any
int vlan 50
ip access-group 103 out
ip access-group 112 in
ex

access-list 90 permit 192.168.1.0 0.0.0.255 
line vty 0 15
access-class 90 in 
ex
```

---

## CORE-SWITCH2 (C-SW2)

```
en
conf t
hostname C-SW2
ip routing
vtp domain nexus
vtp mode client

username admin privilege 15 secret admin123
username support privilege 1 secret support123
ip domain-name nexus.local
crypto key gen rsa general-keys mod 2048

line console 0
login local
exit
enable algorithm-type scrypt secret cisco-nexus-123

line vty 0 15
password cisco-nexus-123
transport input ssh
login local
exit
service password-encryption

int range g1/0/6-7
switchport
channel-group 1 mode active
channel-protocol lacp
ip arp inspection trust
ip dhcp snooping trust

int port-channel 1
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60
ip arp inspection trust
ip dhcp snooping trust

int vl 10
no shut
ip add 192.168.1.3 255.255.255.0
standby 10 priority 100
standby 10 ip 192.168.1.1 
ip helper-address 192.168.3.4

int vl 20
no shut
ip add 192.168.2.3 255.255.255.0
standby 20 priority 100
standby 20 ip 192.168.2.1 
ip helper-address 192.168.3.4

int vl 30
no shut
ip add 192.168.3.3 255.255.255.0
standby 30 priority 100
standby 30 ip 192.168.3.1 

int vl 40
no shut
ip add 192.168.4.3 255.255.255.0
standby 40 priority 110
standby 40 ip 192.168.4.1
standby 40 preempt 
ip helper-address 192.168.3.4

int vl 50
no shut
ip add 192.168.5.3 255.255.255.0
standby 50 priority 110
standby 50 ip 192.168.5.1
standby 50 preempt
ip helper-address 192.168.3.4

int g1/0/1
no switchport 
no shut
ip add 10.0.0.5 255.255.255.252
ex

router ospf 15
router-id 2.1.2.1
network 192.168.0.0 0.0.7.255 area 0
network 10.0.0.4 0.0.0.3 area 0
ex

ip route 0.0.0.0 0.0.0.0 10.0.0.6

int range g1/0/2-5
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60
ip arp inspection trust
ip dhcp snooping trust
ex

spanning-tree vlan 10,20,30 root secondary
spanning-tree vlan 40,50,60 root primary
spanning-tree mode  rapid-pvst

ip dhcp snooping 
ip dhcp snooping vlan 10,20,30,40,50,60
no ip dhcp snooping information option 
ip arp inspection vlan 10,20,30,40,50,60
ip arp inspection validate dst-mac ip src-mac

access-list 100 permit icmp 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 100 deny icmp 192.168.0.0 0.0.7.255 192.168.3.0 0.0.0.255
access-list 100 permit ip any any

access-list 120 permit icmp 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 120 deny icmp 192.168.3.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 120 permit ip any any
int vlan 30
ip access-group 100 out
ip access-group 120 in
ex

access-list 101 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 permit ip 192.168.3.0 0.0.0.255 192.168.2.0 0.0.0.255
access-list 101 deny ip 192.168.0.0 0.0.7.255 192.168.2.0 0.0.0.255
access-list 101 permit ip any any

access-list 110 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 110 permit ip 192.168.2.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 110 deny ip 192.168.2.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 110 permit ip any any
int vlan 20
ip access-group 101 out
ip access-group 110 in
ex

access-list 102 permit ip 192.168.1.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 102 permit ip 192.168.3.0 0.0.0.255 192.168.4.0 0.0.0.255
access-list 102 deny ip 192.168.0.0 0.0.7.255 192.168.4.0 0.0.0.255
access-list 102 permit ip any any

access-list 111 permit ip 192.168.4.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 111 permit ip 192.168.4.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 111 deny ip 192.168.4.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 111 permit ip any any
int vlan 40
ip access-group 102 out
ip access-group 111 in
ex

access-list 103 permit ip 192.168.1.0 0.0.0.255 192.168.5.0 0.0.0.255
access-list 103 permit ip 192.168.3.0 0.0.0.255 192.168.5.0 0.0.0.255
access-list 103 deny ip 192.168.0.0 0.0.7.255 192.168.5.0 0.0.0.255
access-list 103 permit ip any any

access-list 112 permit ip 192.168.5.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 112 permit ip 192.168.5.0 0.0.0.255 192.168.3.0 0.0.0.255
access-list 112 deny ip 192.168.5.0 0.0.0.255 192.168.0.0 0.0.7.255
access-list 112 permit ip any any
int vlan 50
ip access-group 103 out
ip access-group 112 in
ex

access-list 90 permit 192.168.1.0 0.0.0.255
line vty 0 15
access-class 90 in 
ex
```

---

## ACCESS-SWITCH 1 to 4: shared baseline template

Applied to all four access switches before the per-switch VLAN assignments below.

```
en
conf t
hostname <A-SW1-4>
int range g0/1-2
switchport mode trunk
ex
vtp domain nexus
vtp mode client
line console 0
password cisco-nexus
login
exit
enable secret cisco-nexus-123
line vty 0 15
password cisco-nexus-123 
login
exit
service password-encryption

ip dhcp snooping 
ip dhcp snooping vlan 10,20,30,40,50,60
no ip dhcp snooping information option 
ip arp inspection vlan 10,20,30,40,50,60
ip arp inspection validate dst-mac ip src-mac

int range g0/1-2
switchport mode trunk 
switchport trunk allowed vlan 10,20,30,40,50,60
ip arp inspection trust
ip dhcp snooping trust

int range f0/1-4
switchport mode access 
switchport port-security
switchport port-security maximum 2
switchport port-security mac-address sticky
switchport port-security violation shutdown
spanning-tree portfast
spanning-tree bpduguard enable
exit
```

### ACCESS-SWITCH1 (A-SW1)

```
int range f0/1-2
switchport mode access
switchport access vlan 30
int range f0/3-4
switchport mode access
switchport access vlan 10
int f0/1
ip dhcp snooping trust
```

### ACCESS-SWITCH2 (A-SW2)

```
int range f0/1-2
switchport mode access
switchport access vlan 20
int range f0/3-4
switchport mode access
switchport access vlan 50
```

### ACCESS-SWITCH3 (A-SW3)

```
int range f0/1-2
switchport mode access
switchport access vlan 40
int range f0/3-4
switchport mode access
switchport access vlan 20
```

### ACCESS-SWITCH4 (A-SW4)

```
int range f0/1-2
switchport mode access
switchport access vlan 50
int range f0/3-4
switchport mode access
switchport access vlan 40
```

---

## PERIMETER FIREWALL (ASA0, Cisco ASA 5506)

```
en
conf t
int g1/1
no shut 
security-level 100
nameif INTERNAL-CSW1
ip address 10.0.0.2 255.255.255.252
int g1/2
no shut
security-level 100
nameif INTERNAL-CSW2
ip address 10.0.0.6 255.255.255.252
int g1/3
no shut 
security-level 50 
nameif DMZ
ip address 20.0.0.1 255.255.255.0
ex
int g1/4
no shut
security-level 0
nameif OUTSIDE
ip address 10.0.0.9 255.255.255.252
ex

route OUTSIDE 0.0.0.0 0.0.0.0 10.0.0.10 

router ospf 15 
network 10.0.0.0 255.255.255.252 area 0
network 10.0.0.4 255.255.255.252 area 0
network 20.0.0.0 255.255.255.0 area 0

access-list INTERNAL_IN extended permit icmp 192.168.1.0 255.255.255.0 host 10.0.0.2
access-list INTERNAL_IN extended permit icmp 192.168.1.0 255.255.255.0 host 10.0.0.6
access-list INTERNAL_IN extended permit icmp 192.168.1.0 255.255.255.0 host 20.0.0.18
access-list INTERNAL_IN extended permit icmp 192.168.1.0 255.255.255.0 host 20.0.0.12
access-list INTERNAL_IN extended permit icmp 192.168.1.0 255.255.255.0 host 20.0.0.5
access-list INTERNAL_IN extended permit tcp 192.168.0.0 255.255.248.0 host 20.0.0.18 eq 80
access-list INTERNAL_IN extended permit tcp 192.168.0.0 255.255.248.0 host 20.0.0.18 eq 443
access-list INTERNAL_IN extended permit tcp 192.168.0.0 255.255.248.0 host 20.0.0.12 eq 20
access-list INTERNAL_IN extended permit tcp 192.168.0.0 255.255.248.0 host 20.0.0.12 eq 21
access-list INTERNAL_IN extended permit tcp 192.168.0.0 255.255.248.0 host 20.0.0.5 eq 53
access-list INTERNAL_IN extended permit udp 192.168.0.0 255.255.248.0 host 20.0.0.5 eq 53
access-list INTERNAL_IN extended deny ip 192.168.0.0 255.255.248.0 20.0.0.0 255.255.255.0
access-list INTERNAL_IN extended permit ip 192.168.0.0 255.255.248.0 any
access-list INTERNAL_IN extended deny ip any any
access-group INTERNAL_IN in interface INTERNAL-CSW1
access-group INTERNAL_IN in interface INTERNAL-CSW2

access-list DMZ_IN extended permit tcp host 20.0.0.18 192.168.0.0 255.255.248.0
access-list DMZ_IN extended permit tcp host 20.0.0.5 192.168.0.0 255.255.248.0
access-list DMZ_IN extended permit tcp host 20.0.0.12 192.168.0.0 255.255.248.0
access-list DMZ_IN extended permit udp host 20.0.0.5 192.168.0.0 255.255.248.0
access-list DMZ_IN extended permit icmp host 20.0.0.18 192.168.1.0 255.255.255.0
access-list DMZ_IN extended permit icmp host 20.0.0.12 192.168.1.0 255.255.255.0
access-list DMZ_IN extended permit icmp host 20.0.0.5 192.168.1.0 255.255.255.0
access-list DMZ_IN extended deny ip 20.0.0.0 255.255.255.0 192.168.0.0 255.255.248.0
access-list DMZ_IN extended permit ip 20.0.0.0 255.255.255.0 any
access-list DMZ_IN extended deny ip any any
access-group DMZ_IN in interface DMZ

access-list OUTSIDE_IN extended permit tcp any host 20.0.0.18 eq 80
access-list OUTSIDE_IN extended permit tcp any host 20.0.0.18 eq 443
access-list OUTSIDE_IN extended permit tcp any host 20.0.0.12 eq 20
access-list OUTSIDE_IN extended permit tcp any host 20.0.0.12 eq 21
access-list OUTSIDE_IN extended permit tcp any host 20.0.0.5 eq 53
access-list OUTSIDE_IN extended permit udp any host 20.0.0.5 eq 53
access-list OUTSIDE_IN extended deny ip any any
access-group OUTSIDE_IN in interface OUTSIDE

object network INTERNAL-VLANS
subnet 192.168.0.0 255.255.248.0
nat (INTERNAL-CSW1,OUTSIDE) dynamic interface
ex
object network INTERNAL-VLANS-1
subnet 192.168.0.0 255.255.248.0
nat (INTERNAL-CSW2,OUTSIDE) dynamic interface
ex
object network DMZ 
subnet 20.0.0.0 255.255.255.0
nat (DMZ,OUTSIDE) dynamic interface
```

---

## Notes on this configuration set

- ASA0 is a Cisco ASA 5506. Router0 and the DMZ server (Web, FTP, DNS) configurations are not included here, since they weren't supplied alongside the switch/firewall configs, so they aren't reproduced in this file.
- Several discrepancies between what's configured here and the design intent described in `network-documentation.md` are called out in that file's "Open items identified during configuration review" section. Worth checking before treating this config set as final. In short: the floating static routes have no AD set, the ASA advertises the DMZ subnet into OSPF, no loopback interfaces actually exist despite `router-id` referencing loopback-style addresses, and the VTY ACL (90) is a standard ACL rather than the more granular extended ACL the design narrative implies.
