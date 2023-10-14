<!--
SPDX-License-Identifier: Apache-2.0
-->

# Access-LB and Core-LB

These 2 load balancers actually are used for load balancing the data of user equipment and give it to the responsible UPFs. Access-LB load balances the GTP-U encapsulated traffic which are generated by user equipment and encapsulated by gNB and destined to the UPFs, meanwhile the Core-LB load balances the UE traffic which is decapsulated by UPFs and should be sent toward the internet. These two components are built with two Kubernetes resources which are Pod and Service. Like other components their Pod contains a container which is running an app written in Golang. These apps run an HTTP Server, which is responsible for handling requests from UPFs. UPFs may send two kinds of requests to Access-LB and Core-LB:

1.	when a UPF is deployed, register itself to Access-LB and Core-LB through HTTP POST request sending some info which are used to correctly route packets of user equipment toward UPFs. So, in this way Access-LB and Core-LB know how many active UPFs there are.

2.	when a UPF receives a Session Establishment Request, send the IP address of associated user equipment to Access-LB and Core-LB. in this way Access-LB and Core-LB know each UE is assigned to which UPF, so it adds an iptables rule for that specific UE to mark the packets of that user equipment. This iptables rule matches UE’s IP with u32 extension. The reason of using u32 extension is that the data of UE is sent from gNB to UPF in GTP-U encapsulated form, and iptables does not have any direct method to match based on elements of GTP-U header. So, with u32 we can match specific bytes of the header with a certain value which is IP of user equipment. Action of these iptables rules is just adding a mark to the packets that are used to correctly routing the packet using Policy-Based Routing.

### Config of Access-LB for each interface

```
$ echo 101 lb-UPF101 >> /etc/iproute2/rt_tables
$ IP rule add fwmark 101 table lb-UPF101
$ IP route add 192.168.252.3 dev UPF101 table lb-UPF101
$ iptables -t mangle -A PREROUTING -d 192.168.252.3 -p udp --dport 2152 -m u32 --u32 "56&0xffffffff=0xacfa0001" -j MARK --set-mark 101
```

* lb-UPF101: rt_table name
* UPF101: interface of Access-LB connected to UPF101
* Bytes 56-59 contains UE’s IP address
* ACFA0001 is HEX of 172.250.0.1