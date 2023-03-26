# bind9-dns-conf
Example bind9/named configuration to build primary and secondary DNS server

If DNS server is installed on docker (Ubuntu 22.04), please edit the config file of systemd-resolved, /etc/systemd/resolved.conf.

Uncomment the line DNSStubListener, and set it to no.

```
[Resolve]
...
DNSStubListener=no
...
```
Restart the sytemd-resolved service using `sudo systemctl restart systemd-resolved`.

## Primary-DNS Configuration
Bind9 installed on Ubuntu OS 22.04

hostname - ip - domain_name : ns1 - 192.168.100.10/24 - home.arpa

1. Install bind9 packages
```
apt install bind9 bind9utils dnsutils
```
2. Edit /etc/bind/named.conf.options
```
options {
        directory "/var/lib/bind";
        listen-on { 192.168.100.10; };
        allow-query { any; };
        recursion no;

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

...
        dnssec-validation auto;

        listen-on-v6 { any; };
};

logging {
     channel default_debug {
     file "/var/log/named/named.run" versions 2 size 1M;
     severity dynamic; print-time yes;
     };

     channel log_file {
     file "/var/log/named/named.log" versions 2 size 1M;
     severity info; print-time yes; };
     category queries { log_file; };
     };

include "/etc/bind/named.zones";
```
Please refer to /etc/apparmor.d/usr.sbin.named for default directory setting of bind9 logging and zones files

3. Create named.zones on /etc/bind
```
acl "slaves" {
          192.168.100.11;
};

// We are master server for home.arpa
zone "home.arpa" {
        masterfile-format text;
     type master;
     file "home.arpa.db";
     // IP addresses of slave servers allowed to transfer
     allow-transfer { slaves;
     };
};

// We are a master server for reverse zone
zone "100.168.192.in-addr.arpa" {
        masterfile-format text;
     type master;
     file "100.168.192-addr.arpa.db";
     // IP addresses of slave servers allowed to transfer example.com
     allow-transfer { slaves;
     };
};
```
## Secondary-DNS Configuration
Bind9 installed as a container on docker
hostname - ip - domain_name : ns2 - 192.168.100.11 - home.arpa

1. Install docker and docker-compose
```
apt install docker.io docker-compose
```
2. Since bind9 installed as a container please uncomment the line DNSStubListener /etc/systemd/resolved.conf.

Uncomment the line DNSStubListener, and set it to no.

```
[Resolve]
...
DNSStubListener=no
...
```
3. Restart the sytemd-resolved service using `sudo systemctl restart systemd-resolved`.

4. To be tidy, create new directory and create the docker-compose file
```
mkdir ns2-container && touch ns-container/docker-compose.yml
```
5. Edit docker-compose.yml
```
version: "3"

services:
  bind9:
    container_name: ns2-demo-1
    image: ubuntu/bind9
    environment:
      - BIND9_USER=root
      - TZ=UTC
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./config:/etc/bind
      - ./cache:/var/cache/bind
      - ./records:/var/lib/bind
      - ./log:/var/log/named
    restart: unless-stopped
```
6. Create directory config and create named.conf file
```
mkdir ns2-container/config && touch ns-container/config/named.conf
```
7. Edit named.conf file
```
options {
     directory "/var/lib/bind";      // Working directory of zones records
     allow-query { any; };       // This is the default
     recursion no;               // Do not provide recursive service
     # listen-on { any; };
     dnssec-validation auto;

     # /* Path to ISC DLV key */
     # bindkeys-file "/etc/bind/bind.keys";
     # managed-keys-directory "/etc/bind";

};

logging {
     channel default_debug {
     file "/var/log/named/named.run" versions 2 size 1M;
     severity dynamic; print-time yes;
     };

     channel log_file {
     file "/var/log/named/named.log" versions 2 size 1M;
     severity info; print-time yes; };
     category queries { log_file; };
     };
     
// We are a master server for forward zone home.arpa
zone "home.arpa" {
     type slave;
     file "home.arpa.db";
       masters { 192.168.100.10;
     };
     // IP addresses of slave servers allowed to transfer
     allow-transfer {
          192.168.100.11;                       // See spreadsheet, list all slaves
     };
};

// We are a master server for reverse zone
zone "100.168.192.in-addr.arpa" {
     type slave;
     file "100.168.192-addr.arpa.db";
      masters { 192.168.100.10;
     };
     // IP addresses of slave servers allowed to transfer example.com
     allow-transfer {
          192.168.100.11;                       // See spreadsheet, list all slaves
     };
};
```
8. Start the container
```
docker-compose up -d
```
## Test Bind9
```
nslookup ns1 192.168.100.10
nslookup ns2 192.168.100.11
```
