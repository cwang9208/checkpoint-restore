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