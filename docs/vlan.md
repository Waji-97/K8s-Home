## VLAN & Bridging in Linux (removed)
I wanted a seperate network for my kubevirt VMs so I had to create a VLAN in my main PC.
As my PC is using Ubuntu, I edited the file under `/etc/netplan` and added the following lines under the network config.

```bash
  vlans:
    vlan100:
      id: 100
      link: eth0
      dhcp4: no

  bridges:
    br0:
      interfaces: [vlan100]
      addresses: [192.168.10.1/24]
      dhcp4: no
```

Then after `netplan apply`, we can see the new bridge and VLAN
```bash
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ec:9a:0c:18:4d:b3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.1/24 brd 192.168.10.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::b8eb:a9ff:febb:6fdf/64 scope link 
       valid_lft forever preferred_lft forever
4: vlan100@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether ec:9a:0c:18:4d:b3 brd ff:ff:ff:ff:ff:ff
```

But DHCP is disabled and if I want to test Cluster Autoscaling later on, I would need DHCP for automatic IP assignment to my VMs. 
So I decided to install a dhcp server
```bash
sudo apt install isc-dhcp-server
```

Then edit the `/etc/dhcp/dhcpd.conf` file (add the following lines at the end)
```bash
subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option domain-name-servers <nameserver-addresses> (separated with comma);
}
```

Finally, in the `/etc/default/isc-dhcp-server` file, update the interface setting
```bash
INTERFACESv4="br0"
```

We can now enable the dhcp-server
```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

The status
```bash
sudo systemctl status isc-dhcp-server
● isc-dhcp-server.service - ISC DHCP IPv4 server
     Loaded: loaded (/lib/systemd/system/isc-dhcp-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-04-30 11:19:39 UTC; 3s ago
       Docs: man:dhcpd(8)
```

We can also verify bridge
```bash
brctl show
bridge name	   bridge id		     STP enabled	  interfaces
br0		       8000.ec9a0c184db3	 no		          vlan100
```

## VLAN aware Bridge (removed)
```bash
root@pc:/etc/systemd/network# sudo systemctl enable systemd-networkd
Created symlink /etc/systemd/system/dbus-org.freedesktop.network1.service → /lib/systemd/system/systemd-networkd.service.
root@pc:/etc/systemd/network# sudo systemctl restart systemd-networkd
root@pc:/etc/systemd/network# sudo systemctl status systemd-networkd
● systemd-networkd.service - Network Service
     Loaded: loaded (/lib/systemd/system/systemd-networkd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-04-30 15:42:45 UTC; 4s ago
TriggeredBy: ● systemd-networkd.socket
       Docs: man:systemd-networkd.service(8)


root@pc:/etc/systemd/network# pwd
/etc/systemd/network
root@pc:/etc/systemd/network# ll
total 20
drwxr-xr-x 2 root root 4096 Apr 30 15:42 ./
drwxr-xr-x 7 root root 4096 Apr  9 11:56 ../
-rw-r--r-- 1 root root   30 Apr 30 15:42 br0.netdev
-rw-r--r-- 1 root root   70 Apr 30 15:42 br0.network
-rw-r--r-- 1 root root   62 Apr 30 15:42 eth0.network


root@pc:/etc/systemd/network# cat eth0.network 
[Match]
Name=eth0

[Network]
Bridge=br0

[BridgeVLAN]
VID=100
root@pc:/etc/systemd/network# cat br0.netdev 
[NetDev]
Name=br0
Kind=bridge
root@pc:/etc/systemd/network# cat br0.network 
[Match]
Name=br0

[Network]
Address=192.168.10.1/24
VLANFiltering=yes


51: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 9e:41:15:93:f6:81 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.1/24 brd 192.168.10.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::9c41:15ff:fe93:f681/64 scope link 
       valid_lft forever preferred_lft forever
root@pc:/etc/systemd/network# 

```