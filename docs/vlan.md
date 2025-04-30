## VLAN & Bridging in Linux
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
      addresses: [172.32.10.1/24]
      dhcp4: no
```

Then after `netplan apply`, we can see the new bridge and VLAN
```bash
43: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ec:9a:0c:18:4d:b3 brd ff:ff:ff:ff:ff:ff
    inet 172.32.10.1/24 brd 172.32.10.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::f86d:38ff:feb6:2db6/64 scope link 
       valid_lft forever preferred_lft forever
44: vlan100@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br0 state UP group default qlen 1000
    link/ether ec:9a:0c:18:4d:b3 brd ff:ff:ff:ff:ff:ff
```

But DHCP is disabled and if I want to test Cluster Autoscaling later on, I would need DHCP for automatic IP assignment to my VMs. 
So I decided to install a dhcp server
```bash
sudo apt install isc-dhcp-server
```

Then edit the `/etc/dhcp/dhcpd.conf` file (add the following lines at the end)
```bash
subnet 172.32.10.0 netmask 255.255.255.0 {
  range 172.32.10.100 172.32.10.200;
  option routers 172.32.10.1;
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