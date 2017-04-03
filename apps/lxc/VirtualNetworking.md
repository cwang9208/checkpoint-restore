## Virtual network switches
Firstly, libvirt uses the concept of a virtual network switch.

![virtual network switch by itself](images/Virtual_network_switch_by_itself.png)

This is a simple software construction on a host server, that your virtual machines "plug in" to, and direct their traffic through.

![host with a virtual network switch and two guests](images/Host_with_a_virtual_network_switch_and_two_guests.png)

On a Linux host server, the virtual network switch shows up as a network interface.

The default one, created when the libvirt daemon is first installed and started, shows up as virbr0.

![linux host with only a virtual network switch](images/Linux_host_with_only_a_virtual_network_switch.png)

If you're familiar with the `ifconfig` command, you can use that to show it:
```
$ ifconfig virbr0
virbr0    Link encap:Ethernet  HWaddr 3a:d7:6d:e7:4c:67  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
### Network Address Translation (NAT)
By default, a virtual network switch operates in NAT mode.

This means any guests connected through it, use the host IP address for communication to the outside world. Computers external to the host can't initiate communications to the guests inside, when the virtual network switch is operating in NAT mode.

![host with a virtual network switch in nat mode and two guests](images/Host_with_a_virtual_network_switch_in_nat_mode_and_two_guests.png)

### Forwarding Incoming Connections
By default, guests that are connected via a virtual network with NAT forwarding can make any outgoing network connection they like. Incoming connections are allowed from the host, and from other guests connected to the same libvirt network, but all other incoming connections are blocked by iptables rules.

If you would like to make a service that is on a guest behind a NATed virtual network publicly available, you can install the necessary iptables rules to forward incoming connections to the host on any given port HP to port GP on the guest GNAME:

1\) Determine a) the name of the guest "G" (as defined in the libvirt domain XML), b) the IP address of the guest "I", c) the port on the guest that will receive the connections "GP", and d) the port on the host that will be forwarded to the guest "HP".

Use the basic script below (port mappings)
```
#!/bin/bash

# Update the following variables to fit your setup
GUEST_IP=
GUEST_PORT=
HOST_PORT=

iptables -I FORWARD -p tcp -d  $GUEST_IP -j ACCEPT
iptables -t nat -I PREROUTING -p tcp --dport $HOST_PORT -j DNAT --to $GUEST_IP:$GUEST_PORT
```
## Bridged networking
网桥(bridge)模式可以让客户机和宿主机共享一个物理网络设备连接，客户机有自己的独立IP地址，可以直接连接与宿主机一模一样的网络，客户机可以访问外部网络，外部网络也可以直接访问客户机（就像普通物理主机一样）。
