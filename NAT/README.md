# NAT !!!

This document is just a handy command keeper for NAT! :)

```shell

iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
iptables --append FORWARD --in-interface eth1 -j ACCEPT
apt update -y && apt install iptables-persistent
iptables-save > /etc/iptables/rules.v4

```
