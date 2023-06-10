# bind9
# Installation

To begin the installation process, open the terminal and execute the following commands:

```shell
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```

# Primary Server Configuration
#### named.conf.options

Create a new ACL named "trusted" above the "options" section:

```shell
sudo nano /etc/bind/named.conf.options
```

```shell
acl "trusted" {
    192.168.222.16;    # ns1
    192.168.222.17;    # ns2
    192.168.222.20;    # host1
    192.168.222.21;    # host2
};

options {
    directory "/var/cache/bind";
    recursion yes;                 # enables recursive queries
    allow-recursion { trusted; };  # allows recursive queries from "trusted" clients
    listen-on { 192.168.222.16; }; # ns1 private IP address - listen on private network only
    allow-transfer { none; };      # disable zone transfers by default

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    . . .
};
```

#### named.conf.local

Edit the "named.conf.local" file:

```shell
sudo nano /etc/bind/named.conf.local
```

Primary Zone
```shell
zone "dns.eibanez.cf" {
    type primary;
    file "/etc/bind/zones/db.dns.eibanez.cf";    # zone file path
    allow-transfer { 192.168.222.17; };           # ns2 private IP address - secondary
};
```

Reverse Zone
```shell
zone "222.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.222.168.192";       # 192.168.222.0/24 subnet
    allow-transfer { 192.168.222.17; };           # ns2 private IP address - secondary
};

```

### Zone Files

#### /etc/bind/zones Directory

Create the directory:

```shell
sudo mkdir /etc/bind/zones
```
Forward Zone File
Copy the local file to the zone directory:
```shell
sudo cp /etc/bind/db.local /etc/bind/zones/db.dns.eibanez.cf
```
Edit the zone file:
```shell
sudo nano /etc/bind/zones/db.dns.eibanez.cf
```
Define the Start of Authority (SOA) using the FQDN of your primary DNS server (ns1). Increment the serial value +1:
```shell
;
$TTL    604800
@       IN      SOA     ns1.dns.eibanez.cf. admin.dns.eibanez.cf. (
                              3         ; Serial
```
Specify the NS records that point to your primary and secondary DNS servers:
```shell
; name servers - NS records
IN      NS      ns1.dns.eibanez.cf.
IN      NS      ns2.dns.eibanez.cf.
```
Add the A records for your DNS servers and clients:
```shell
; name servers - A records
ns1.dns.eibanez.cf.     IN      A       192.168.222.16
ns2.dns.eibanez.cf.     IN      A       192.168.222.17

; 192.168.222.0/24 - A records
dnsclient1.dns.eibanez.cf.        IN      A      192.168.222.20
dnsclient2.dnz.eibanez.cf.        IN      A      192.168.222.21
```

#### Reverse Zone File

Copy the example file to create the reverse zone:

```shell
sudo cp /etc/bind/db.127 /etc/bind/zones/db.222.168.192
```

Edit the reverse zone file:
```shell
sudo nano /etc/bind/zones/db.222.168.192
```

Increment the serial value in the SOA record:
```shell
@       IN      SOA     ns1.dns.eibanez.cf. admin.dns.eibanez.cf. (
                              3         ; Serial
```

Add the NS records for your DNS servers:
```shell
; name servers - NS records
IN      NS      ns1.dns.eibanez.cf.
IN      NS      ns2.dns.eibanez.cf.
```

Add the PTR records for the IP addresses in your zone (192.168.222.0/24):
```shell
; PTR Records
16      IN      PTR     ns1.dns.eibanez.cf.          ; 192.168.222.16
17      IN      PTR     ns2.dns.eibanez.cf.          ; 192.168.222.17
20      IN      PTR     dnsclient1.dns.eibanez.cf.   ; 192.168.222.20
21      IN      PTR     dnsclient2.dns.eibanez.cf.   ; 192.168.222.21
```

### Final Configuration

To ensure the configuration files are correct, run the following commands:

```shell
sudo named-checkconf
sudo named-checkzone dns.eibanez.cf /etc/bind/zones/db.dns.eibanez.cf
sudo named-checkzone 222.168.192.in-addr.arpa /etc/bind/zones/db.222.168.192
```
Restart the BIND service:

```shell
sudo systemctl restart bind9
```

Allow Bind9 through UFW Firewall:
```shell
sudo ufw allow Bind9
```
And you should be done!

# Secondary Server Configuration
#### named.conf.options

Edit the "named.conf.options" file:

```shell
sudo nano /etc/bind/named.conf.options
```

Add the "trusted" ACL definition:
```shell
acl "trusted" {
    192.168.222.16;    # ns1
    192.168.222.17;    # ns2
    192.168.222.20;    # host1
    192.168.222.21;    # host2
};

options {
    directory "/var/cache/bind";
    recursion yes;                 # enables recursive queries
    allow-recursion { trusted; };  # allows recursive queries from "trusted" clients
    listen-on { 192.168.222.17; }; # ns2 private IP address - listen on private network only
    allow-transfer { none; };      # disable zone transfers by default

    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
};
```

#### named.conf.local

Edit the "named.conf.local" file:

```shell
sudo nano /etc/bind/named.conf.local
```

Add the following configurations:
Primary Zone
```shell
zone "dns.eibanez.cf" {
    type slave;
    file "db.dns.eibanez.cf";
    masters { 192.168.222.16; };
};
```

Reverse Zone
```shell
zone "222.168.192.in-addr.arpa" {
    type slave;
    file "db.222.168.192";
    masters { 192.168.222.16; };
};
```
#### Final Configuration

To ensure the configuration files are correct, run the following commands:
```shell
sudo named-checkconf
sudo named-checkzone dns.eibanez.cf /etc/bind/db.dns.eibanez.cf
sudo named-checkzone 222.168.192.in-addr.arpa /etc/bind/db.222.168.192
```

Restart the BIND service:
```shell
sudo systemctl restart bind9
```

Allow Bind9 through UFW Firewall:
```shell
sudo ufw allow Bind9
```

### Client Configuration

Show IP addresses for the subnet 192.168.222.0/24:

```shell
ip a show to 192.168.222.0/24
```

Now you know your network interface's name, in my case it's **ens32**

Modify the netplan configuration file:
```shell
sudo nano /etc/netplan/00-installer-config.yaml
```

With the following data
```shell
network:
  version: 2
  ethernets:
    eth1:                                    # Private network interface
      nameservers:
        addresses:
          - 192.168.222.16                     # Private IP for ns1
          - 192.168.222.17                     # Private IP for ns2
      search: [ dns.eibanez.cf ]               # DNS zone
```

Test the configuration:
```shell
sudo netplan try
```
Accept the changes.

Check the DNS configuration with the following command:
```shell
resolvectl status
```

Now you can use the commands nslookup, dig and host to check that the resolve is getting done properly
```shell
nslookup ns1
```
```shell
nslookup 192.168.222.16
```
