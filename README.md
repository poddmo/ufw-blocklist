# ufw-blocklist
Add IP blocklists to ufw, the Ubuntu firewall
* integrates into ufw for pure Ubuntu
* uses Linux ipsets for performance
* the blocklist is refreshed daily
* blocks inbound, outbound and forwarding packets
* tested on:
  * Armbian 22.05.3 Focal (based on Ubuntu 20.04.4 LTS (Focal Fossa))
  * Ubuntu 22.04 LTS
* IP blocklist sourced from [IPsum](https://github.com/stamparm/ipsum)

# Install
Install ipset package
```
sudo apt install ipset
```

Install ufw-blocklist files:
* /etc/ufw/after.init
* /etc/cron.daily/ufw-blocklist-ipsum
```
chmod 755 after.init ufw-blocklist-ipsum
cp after.init /etc/ufw/after.init
cp ufw-blocklist-ipsum /etc/cron.daily/ufw-blocklist-ipsum
```

Download an initial IP blocklist:
```
curl -sS -f --compressed 'https://raw.githubusercontent.com/stamparm/ipsum/master/levels/4.txt' > /etc/ipsum.4.txt
```
Restart ufw
```
ufw reload
```

# Usage
The blocklist is automatically started and stopped by ufw. There are 2 additional commands available: status and flush-all
- The status option is described in the Monitor section below.
- The flush-all option deletes all entries in the blocklist and zeros the iptables hit counters. Use /etc/cron.daily/ufw-blocklist-ipsum to download the latest list and repopulate the ipset. Alternately, IP addresses can be manually added to the ipset like this:
```
ipset add ufw-blocklist-ipsum a.b.c.d
```

# Monitor
Calling after.init with the status option displays the current count of the entries in the blocklist, the hit counts on the firewall rules (column 1 is hits, column 2 is bytes) and log messages:
```
user@ubunturouter:~# sudo /etc/ufw/after.init status
Name: ufw-blocklist-ipsum
Type: hash:net
Revision: 6
Header: family inet hashsize 4096 maxelem 65536
Size in memory: 357312
References: 3
Number of entries: 12789
   76998  4403836 ufw-blocklist-input  all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set ufw-blocklist-ipsum src
       4      160 ufw-blocklist-forward  all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set ufw-blocklist-ipsum dst
      11      868 ufw-blocklist-output  all  --  *      *       0.0.0.0/0            0.0.0.0/0            match-set ufw-blocklist-ipsum dst
Sep 24 06:25:01 ubunturouter ufw-blocklist-ipsum[535172]: starting update of ufw-blocklist-ipsum with 12654 entries from https://raw.githubusercontent.com/stamparm/ipsum/master/levels/3.txt
Sep 24 06:26:02 ubunturouter ufw-blocklist-ipsum[547387]: finished updating ufw-blocklist-ipsum. Old entry count: 12654 New count: 12181 of 12181
Sep 24 22:23:21 ubunturouter kernel: [UFW BLOCKLIST FORWARD] IN=eth1 OUT=ppp0 MAC=11:22:33:44:55:66:77:88:99:00:aa:bb:cc:dd SRC=192.168.1.11 DST=194.165.16.37 LEN=40 TOS=0x00 PREC=0x00 TTL=62 ID=0 DF PROTO=TCP SPT=51413 DPT=65058 WINDOW=0 RES=0x00 ACK RST URGP=0
Sep 25 06:25:02 ubunturouter ufw-blocklist-ipsum[598717]: starting update of ufw-blocklist-ipsum with 12181 entries from https://raw.githubusercontent.com/stamparm/ipsum/master/levels/3.txt
Sep 25 06:26:07 ubunturouter ufw-blocklist-ipsum[611761]: finished updating ufw-blocklist-ipsum. Old entry count: 12181 New count: 13008 of 13008
Sep 25 21:19:42 ubunturouter kernel: [UFW BLOCKLIST FORWARD] IN=eth1 OUT=ppp0 MAC=11:22:33:44:55:66:77:88:99:00:aa:bb:cc:dd SRC=192.168.1.11 DST=45.227.254.8 LEN=40 TOS=0x00 PREC=0x00 TTL=62 ID=0 DF PROTO=TCP SPT=51413 DPT=65469 WINDOW=0 RES=0x00 ACK RST URGP=0
Sep 25 21:19:45 ubunturouter kernel: [UFW BLOCKLIST FORWARD] IN=eth1 OUT=ppp0 MAC=11:22:33:44:55:66:77:88:99:00:aa:bb:cc:dd SRC=192.168.1.11 DST=45.227.254.8 LEN=40 TOS=0x00 PREC=0x00 TTL=62 ID=0 DF PROTO=TCP SPT=51413 DPT=65469 WINDOW=0 RES=0x00 ACK RST URGP=0
Sep 25 21:19:51 ubunturouter kernel: [UFW BLOCKLIST FORWARD] IN=eth1 OUT=ppp0 MAC=11:22:33:44:55:66:77:88:99:00:aa:bb:cc:dd SRC=192.168.1.11 DST=45.227.254.8 LEN=40 TOS=0x00 PREC=0x00 TTL=62 ID=0 DF PROTO=TCP SPT=51413 DPT=65469 WINDOW=0 RES=0x00 ACK RST URGP=0
Sep 26 06:25:02 ubunturouter ufw-blocklist-ipsum[661335]: starting update of ufw-blocklist-ipsum with 13008 entries from https://raw.githubusercontent.com/stamparm/ipsum/master/levels/3.txt
Sep 26 06:26:06 ubunturouter ufw-blocklist-ipsum[674158]: finished updating ufw-blocklist-ipsum. Old entry count: 13008 New count: 12789 of 12789
```
- Hits on the OUTBOUND or FORWARD drop rule may indicate an issue with an internal host and are logged. In the example status shown above, the hits on the FORWARD rule are related to an internal torrent client.
- INPUT hits are not logged. The status output above shows **76998 dropped INPUT packets** after the system has been up 9 days, 22:45 hours.

# Todo
These scripts have run flawlessly for 2 years. The next steps will take advantage of this extended ufw-framework to block bogans and create a whitelist - generalising the blocklist case to arbitrary ipsets
- create an after.init.d directory, rename after.init to /etc/ufw/after.init.d/10-ufw-blocklist-ipsum
- restore the original after.init, modify to test if after.init.d exists. If so, use run-parts(8)
- create /etc/ufw/after.init.d/20-ufw-blocklist-bogans for blocking bogan IP addresses
- create /etc/ufw/after.init.d/99-ufw-whitelist-mgt for lockout prevention in case our management IP address makes its way into the blocklists. This script must run last due to the insert rules.
- create pull request to ufw upstream for modified ufw-framework after.init
