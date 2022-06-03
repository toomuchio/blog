## Conserving IPv4 resources by converting /30 subnets to /31 subnets.

At a current cost of ~$48-50 USD per address and dwindling supply, nobody can afford to be wasting them.

I'm not covering the network side of this everybody is different, but needless to say with Arista hardware it was fairly easy to achieve but it's my understanding that MikroTik's can have trouble with it.

It's probably worth lightly going into why we use /30s and /31s, now yes you can just use a flat style VLAN with a /24 or /28 and conserve even more address space, since you can use a shared gateway.

The trouble with this is opens the door to ARP spoofing, broadcast traffic intercepting, address hijacking, lack of portability (what if I want that "address over" there now) and a plethora of other issues.

---------------

Back to the issue at hand, rolling out /31s was fairly easy most hosts in this situation are Linux and BSD based and I saw no issues, RFC3021 is implemented correctly by Linux and BSD.

... Then I tried ... You guessed it Windows, no matter the version it just refuses to work Server 2012, 2016, 2019, 2022 latest patch levels. Great...

https://social.technet.microsoft.com/Forums/en-US/6da37a2d-6884-4c3c-bdd5-1b8356edfced/windows-102019-non-compliant-with-rfc-3021-ipv4-31-subnet-mask?forum=winserverPN

https://community.spiceworks.com/topic/2111512-isp-31-subnet

Well that's that right? Can't be done right? Well not exactly, by setting the subnet mask 255.255.255.255 what would be for a /32. It does work! You do get some a nasty warning but who cares about that.

So I've found a solution to that, but the issue is our OS install automation depends on PXE with a DHCP server.

Now if you're following along you'll probably realise the issue here, the PXE shim-loader wont see 255.255.255.255 as valid with a /31 so now any PXE boot fails, change it to 255.255.255.254 it works but once the Windows installer boots it wont get internet and installs fail.

Luckily most DHCP servers have a hook/callback once the offered lease has been taken, so the solution here was hacky but simple
- Start with a 255.255.255.254 subnet lease, to get into PXE boot
- If the OS being installed is Windows after the .254 subnet lease is taken, switch to a 255.255.255.255 subnet lease
- Windows SIM XML value also has to be set to a /32 as well obviously e.g. ```<IpAddress wcm:action="add" wcm:keyValue="1">192.168.0.1/32</IpAddress>```

(The Windows installer actually hammers the DHCP server for 5m or so till it gets what it sees as a valid lease before giving up, so I could have detected it that way as well.)

Now everything works! And I've effectively doubled the amount of IP addresses we can use.

Remember a /30 is 4 addresses, a /31 is 2 addresses.
