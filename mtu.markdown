At a remote site a few wireless clients were having intermittent issues. For example, two computers (on the domain) side by side/configured the same way --- one would get L3 connectivity and the other would not grab an IP.

On the Aruba controller, one would show up in `show ap association` and `show user-table`. While the problematic clients would only show up in `show ap association`, this means they're associated at L2, but they don't have L3 connectivity.

Analyzing Splunk showed DHCPOFFERs followed by DHCPDISCOVERs in what looked like a loop. I checked the InfoBlox DHCP pools (VLAN pooling), L3 configs and assignments between Nokia PE and Aruba Controllers, and the controller configs. Checks passed.

But then I found that packets were being fragmented from the router to the campus AP:

`size 1473`:

```
A:PE01# ping 10.50.100.99 router 9001 detail do-not-fragment size 1473 
PING 10.50.100.99 1473 data bytes
136 bytes from 10.50.100.2: Frag needed and DF set
VR HL TOS   LEN   ID FLG  OFF TTL PRO  CKS SRC             DST
 4  5  00  1501 1f2d   2 0000  64   1 205b 10.50.100.2     10.50.100.99
ICMP: Echo request
136 bytes from 10.50.100.2: Frag needed and DF set
VR HL TOS   LEN   ID FLG  OFF TTL PRO  CKS SRC             DST
 4  5  00  1501 1f55   2 0000  64   1 2033 10.50.100.2     10.50.100.99  
ICMP: Echo request
```

`size 1472`:

```
A:PE01# ping 10.50.100.99 router 9001 detail do-not-fragment size 1472 
PING 10.50.100.99 1472 data bytes
1480 bytes from 10.50.100.99: icmp_seq=1 ttl=64 time=1.13ms.
1480 bytes from 10.50.100.99: icmp_seq=2 ttl=64 time=0.640ms.
```

The GRE tunnels between controller and APs have a SAP MTU of 1500. This means there is a fragmentation issue on the GRE tunnels. Packets fragmentation generally causes degraded perf and throughput.

The fix is __A:__ fix it up stream, __B:__ fix it on the controller by adjusting the SAP MTU on the current AP system profile, or __C:__ create a new AP system profile and AP group, and moving the APs into it.

I created a new AP system profile and adjusted the SAP MTU to 1450, and created a new AP group. 

```
ap system-profile "new-profile"
   mtu 1400
```

I moved the APs into the new group which means the APs will reboot. Once the APs came back up, clients having intermittent issues immediately associated and got L3 connectivity.

To verify change on the controller:

```
(controller)# show ap bss-table
```
