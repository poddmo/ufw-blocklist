# ufw-ipsum
Add an IP blacklist to ufw
* integrates into ufw for pure ubuntu
* uses ipsets
* IP blacklist is refreshed daily
* blocks inbound and forwarding packets
* tested on:
  * Armbian 22.05.3 Focal (based on Ubuntu 20.04.4 LTS (Focal Fossa))
  * Ubuntu 22.04 LTS
* IP blacklist sourced from [IPsum](https://github.com/stamparm/ipsum)

# Install
Install ipset package
```
apt install ipset
```

Install ufw-ipsum files:
* /etc/ufw/after.init
* /etc/cron.daily/ipsum2ipset
```
cp after.init /etc/ufw/after.init
cp ipsum2ipset /etc/cron.daily/ipsum2ipset
```

The default internet interface is ppp0. Edit ```after.init``` to change it.

Download an initial IP blacklist:
```
curl -sS -f --compressed 'https://raw.githubusercontent.com/stamparm/ipsum/master/levels/4.txt' > /etc/ipsum.4.txt
```
Restart ufw
```
ufw reload
```

# Monitor
Get a count of the number of IP addresses in the active blacklist:
```
root:~# /etc/ufw/after.init status
Name: ipsum
Type: hash:net
Revision: 6
Header: family inet hashsize 1024 maxelem 65536
Size in memory: 122664
References: 2
Number of entries: 4561
```
View the hits to/from blacklist hosts:
```
root:~# iptables -L -nvx | grep ipsum
Chain INPUT (policy DROP 11 packets, 630 bytes)
    pkts      bytes target     prot opt in     out     source               destination
    1202    71161 DROP       all  --  ppp0   *       0.0.0.0/0            0.0.0.0/0            match-set ipsum src
Chain FORWARD (policy DROP 0 packets, 0 bytes)
    pkts      bytes target     prot opt in     out     source               destination
       0        0 ufw-after-ipsum-fwddrop  all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set ipsum dst
```
Hits on the FORWARD drop rule may indicate an issue with an internal host.
