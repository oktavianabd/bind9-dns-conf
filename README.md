# bind9-dns-conf
Example bind9/named configuration to build primary and secondary DNS server

OS: Ubuntu 22.04
Domain Name: home.arpa

Primary-DNS:
hostname: ns1
ip: 192.168.100.10

1. Install bind9 packages
apt install bind9 bind9utils dnsutils

2. Copy named.conf.options and named.zones

3. Copy master "forward and reverse" zone file 

Secondary-DNS:
hostname: ns2
ip: 192.168.100.11

1. Install bind9 packages
apt install bind9 bind9utils dnsutils

2. Copy named.conf.options

3. "forward and reverse" zone file would be copied automatically from primary DNS
