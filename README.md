guard-cartel
============

Small script to manage multi-server Linux iptables in a away similar
to AWS EC2 Security Groups.

Each cartel has several properties:
- unique name
- list of members, IP cidr or other cartels
- list of permissions used for iptables rules

From these properties iptables rules are generated, while denying all other
traffic that is not permitted in the rules.

Example:

```
Server A
- cartel X
- ip 10.0.0.1

Cartel X
- rules:
  - proto tcp port 22 from 0.0.0.0/32
  - proto tcp port 3306 from Cartel X
```

`cartel` is executed and produces the following list of rules on Server A:

```
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:ssh
ACCEPT     tcp  --  10.0.0.1             anywhere            tcp dpt:ssh
REJECT     tcp  --  anywhere             anywhere            tcp flags:SYN,RST,ACK/SYN reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere            reject-with icmp-port-unreachable
```


More complex example:

```
Server A          Server B
- cartel X        - cartel Y
- cartel Y        - ip 10.0.0.2
- ip 10.0.0.1

Cartel X
- rules:
  - proto tcp port 22 from 0.0.0.0/32
  - proto tcp port 80 from Cartel Y
  - proto tcp port 443 from Cartel Y

Cartel Y
- rules:
  - proto tcp port 22 from 0.0.0.0/32
  - proto tcp port 11211 from Cartel X
  - proto tcp port 3306 from Cartel Y
```


`cartel` is executed for Server A produces the following list of rules:

```
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:ssh
ACCEPT     tcp  --  10.0.0.2             anywhere            tcp dpt:http
ACCEPT     tcp  --  10.0.0.2             anywhere            tcp dpt:https
ACCEPT     tcp  --  10.0.0.2             anywhere            tcp dpt:mysql
REJECT     tcp  --  anywhere             anywhere            tcp flags:SYN,RST,ACK/SYN reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere            reject-with icmp-port-unreachable
```

`cartel` is executed for Server B produces the following list of rules:

```
target     prot opt source               destination
ACCEPT     all  --  anywhere             anywhere            state RELATED,ESTABLISHED
ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:ssh
ACCEPT     tcp  --  10.0.0.1             anywhere            tcp dpt:memcached
ACCEPT     tcp  --  10.0.0.1             anywhere            tcp dpt:mysql
REJECT     tcp  --  anywhere             anywhere            tcp flags:SYN,RST,ACK/SYN reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere            reject-with icmp-port-unreachable
```


The server stores the list of his own cartels at `/etc/cartel/identity`, and
the list of cartels at `/etc/cartel/X.members`, `/etc/cartel/X.rules

The `identity` file contains a list of cartel names.
The cartel X.members file contains a list of IPs or other cartel names.
The cartel X.rules file contains a list of rules for that cartel.

These scripts are not responsible for filling these files, but use them to
generate the iptable rules.
