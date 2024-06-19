# ufw-blocklist
Add an IP blocklist to ufw, the uncomplicated Ubuntu firewall
* integrates into ufw for pure Ubuntu
* blocks inbound, outbound and forwarding packets
* uses [Linux ipsets](https://ipset.netfilter.org/) for kernel-grade performance
* the IP blocklist is refreshed daily
* the IP blocklist is sourced from [IPsum](https://github.com/stamparm/ipsum)
* ufw-blocklist is tested on:
  * Armbian 22.05.3 Focal (based on Ubuntu 20.04.4 LTS (Focal Fossa))
  * Ubuntu 22.04 LTS (Jammy Jellyfish)

**This blocklist is _very_ successful at dropping a lot of uninvited traffic.** It has been intentionally designed to be very light on resource requirements and zero maintenance as the initial target platform was a single-board computer operating as a home internet gateway. After the initial installation, there are no further writes to the storage system to preserve solid state storage. I would now highly recommend it for any Ubuntu host that has a public IP address or is otherwise exposed directly to the internet, for example, by port forwarding.

# Installation
Install the ipset package
```
sudo apt install ipset
```

Backup the original ufw `after.init` example script
```
sudo cp /etc/ufw/after.init /etc/ufw/after.init.orig
```

Install the ufw-blocklist files
```
git clone https://github.com/poddmo/ufw-blocklist.git
cd ufw-blocklist
sudo cp after.init /etc/ufw/after.init
sudo cp ufw-blocklist-ipsum /etc/cron.daily/ufw-blocklist-ipsum
sudo chown root:root /etc/ufw/after.init /etc/cron.daily/ufw-blocklist-ipsum
sudo chmod 750 /etc/ufw/after.init /etc/cron.daily/ufw-blocklist-ipsum
```

Download an initial IP blocklist from [IPsum](https://github.com/stamparm/ipsum)
```
curl -sS -f --compressed -o ipsum.4.txt 'https://raw.githubusercontent.com/stamparm/ipsum/master/levels/4.txt'
sudo chmod 640 ipsum.4.txt
sudo cp ipsum.4.txt /etc/ipsum.4.txt
```
Start ufw-blocklist
```
sudo /etc/ufw/after.init start
```
It takes time to load the blocklist entries into the ipset. Watch the progress with 
```
sudo ipset list ufw-blocklist-ipsum -terse | grep 'Number of entries'
```

# Usage
The blocklist is automatically started and stopped by ufw using the enable, disable and reload options. See the [Ubuntu UFW wiki page](https://help.ubuntu.com/community/UFW) for help getting started with ufw.

There are 2 additional `after.init` commands available: status and flush-all
- The **status** option shows the count of entries in the blocklist, the hit count of packets that have been blocked and the last 10 log entries. The status option is further explained in the [Status](#status) section below.
- The **flush-all** option deletes all entries in the blocklist and zeros the iptables hit counters:
```
sudo /etc/ufw/after.init flush-all
```
From this state you can manually add IP addresses to the list like this:
```
sudo ipset add ufw-blocklist-ipsum a.b.c.d
```
This is useful for testing. Use `/etc/cron.daily/ufw-blocklist-ipsum` to download the latest list and fully restore the blocklist.

# Status
Calling `after.init` with the status option displays the current count of the entries in the blocklist, the hit counts on the firewall rules (column 1 is hits, column 2 is bytes) and the last 10 log messages. Here is a sample output:
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
- Hits on the OUTPUT or FORWARD drop rules may indicate an issue with an internal host and are logged. In the example status shown above, the hits on the FORWARD rule are related to an internal torrent client.
- INPUT hits are not logged. The status output above shows **76998 dropped INPUT packets** after the system has been up 9 days, 22:45 hours.

# Todo
These scripts have run flawlessly for 2 years. The next steps will take advantage of this extended ufw-framework and generalise the blocklist case to arbitrary ipsets, for example, to block bogans or by geoblock
- test and document use of after.init_run-parts
- test and document geo-block example for blocking geographic subnets. Geo-based blocks are useful for blocking botnets or "citizen activists." Geo-based subnets can be found at:
  - https://www.ip2location.com/free/visitor-blocker
  - https://www.ipdeny.com/ipblocks/
- test and document blocking bogan IP addresses. Bogon lists can be found at:
  - FireHOL includes fullbogons: https://iplists.firehol.org/
  - so does team Cymru. See fullbogons at: https://www.team-cymru.com/bogon-reference-http
- develop a whitelist an ip/cidr address
- develop test of entries as valid ip/cidr addresses
