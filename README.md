# Bind9 Primary and Secondary DNS Server Configuration Using Docker
Example bind9/named configuration to build primary and secondary DNS server, then secondary DNS server deploy as a container using docker

***!Important, If DNS server is installed on docker (Ubuntu 22.04), please edit the config file of systemd-resolved, /etc/systemd/resolved.conf.***
Uncomment the line DNSStubListener, and set it to no.

```
[Resolve]
...
DNSStubListener=no
...
```
Restart the sytemd-resolved service using `sudo systemctl restart systemd-resolved`.

## Primary-DNS Configuration
Bind9 installed on baremetal Ubuntu OS 22.04 (not using docker)

hostname - ip - domain_name : ns1 - 192.168.100.10/24 - home.arpa

1. Install bind9 packages
```
apt install bind9 bind9utils dnsutils bind9-doc
```
2. Edit /etc/bind/named.conf.options
```
options {
        directory "/var/cache/bind";
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
4. Create /var/log/named directory and change owner to bind
```
mkdir /var/log/named
chown bind /var/log/named
```
5. Create named.zones on /etc/bind
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
6. Check configuration file if there is any error
```
named-checkconf named.conf.options
```
7. Create home.arpa.db on /var/cache/bind
```
$TTL    604800
@       IN      SOA     ns1.home.arpa. admin.home.arpa. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
;
; name servers - NS records
     IN      NS      ns1.home.arpa.
     IN      NS      ns2.home.arpa.

; name servers - A records
ns1.home.arpa.          IN      A       192.168.100.10
ns2.home.arpa.          IN      A       192.168.100.11

; 192.168.100.0/24 - A records
host1.home.arpa.        IN      A      192.168.100.12
host2.home.arpa.        IN      A      192.168.100.13
```
8. Create 100.168.192-addr.arpa.db on /var/cache/bind
```
$TTL    604800
@       IN      SOA     home.arpa. admin.home.arpa. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
      IN      NS      ns1.home.arpa.
      IN      NS      ns2.home.arpa.

; PTR Records
10             IN      PTR     ns1.home.arpa.    ; 192.168.100.110
11             IN      PTR     ns2.home.arpa.    ; 192.168.100.11
12             IN      PTR     host1.home.arpa.  ; 192.168.100.12
13             IN      PTR     host2.home.arpa.  ; 192.168.100.13
```
9. Check zone file if there is any error
```
#named-checkzone home.arpa home.arpa.db
zone home.arpa/IN: loaded serial 3
OK

#named-checkzone 100.168.192-addr.arpa 100.168.192-addr.arpa.db
zone 100.168.192-addr.arpa/IN: loaded serial 3
OK
```
10. Restart bind9 service
```
systemctl restart bind9
```

## Secondary-DNS Configuration
Bind9 installed as a container on docker

hostname - ip - domain_name : ns2 - 192.168.100.11 - home.arpa

1. Install docker and docker-compose
```
apt install docker.io docker-compose
```
2. Since bind9 installed as a container please uncomment the line DNSStubListener on /etc/systemd/resolved.conf.

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
root@ns1:/var/cache/bind# nslookup 192.168.100.10 192.168.100.10
10.100.168.192.in-addr.arpa     name = ns1.home.arpa.

root@ns1:/var/cache/bind# nslookup 192.168.100.11 192.168.100.10
11.100.168.192.in-addr.arpa     name = ns2.home.arpa.

root@ns1:/var/cache/bind# nslookup ns1 192.168.100.10
Server:         192.168.100.10
Address:        192.168.100.10#53

Name:   ns1.home.arpa
Address: 192.168.100.10

root@ns1:/var/cache/bind# nslookup ns2 192.168.100.10
Server:         192.168.100.10
Address:        192.168.100.10#53

Name:   ns2.home.arpa
Address: 192.168.100.11
```
