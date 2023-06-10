# bind9
# Installation

To begin the installation process, open the terminal and execute the following commands:

```shell
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```
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
