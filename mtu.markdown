Some but not all wireless clients were having intermittent issues at a remote site. For example, two domain computers configured the same way, and side by side. One would get an IP, the other would not.

One would show up in both `show ap association` and `show user-table`. And the one without an IP would only show up in `show ap association`. 

Splunk showed DHCPOFFER followed by DHCPDISCOVER over and over. DHCP pool looked fine. L3 networks were correctly built and assigned to the correct ports on the router. The primary/backup controller configs looked great.

Then I found that packets were being fragmented at 1473, but was good at 1472, but fragmented at 1473.

1473:

```
(ctrlr) # ping 10.X.X.161 df-flag packet-size 1473
Press 'q' to abort.
Sending 5, 1473-byte ICMP Echos to 10.X.X.161, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

1472:

```
(ctrlr) # ping 10.X.X.161 df-flag packet-size 1472
Press 'q' to abort.
Sending 5, 1472-byte ICMP Echos to 10.X.X.161, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4.84/5.4892/6.346 ms
```

Solution was to adjust the mtu size under the AP profile:

```
ap system-profile "profile"
   mtu 1400
```

Moved the APs into the new group. APs rebooted. Then all clients having issues immediately connected and grabbed an address.
