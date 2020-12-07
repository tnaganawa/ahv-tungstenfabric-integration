# ahv-tungstenfabric-integration
AHV tungstengfabric vRouter integration (Work in progress)
 - Note: this is my personal attempt to understand curent vRouter capability, not meant for some further effort


## Installation

### Install AHV

Install AHV (and CVM) from Nutanix CE ISO file (Nutanix CE 5.18)
- https://dreadysblog.com/2017/06/29/building-a-hci-lab-with-nutanix-community-edition/

iso file can be downloaded from this site:
- https://next.nutanix.com/discussion-forum-14/download-community-edition-38417

### AHV OVS setting

create OVS bridge, from CVM node:

  ```
allssh manage_ovs --bridge_name bro001 create_single_bridge
  ```

bro001 is created for the similar usecase with CVM vm_dvs
- https://next.nutanix.com/community-blog-154/deploying-contrail-sdn-controller-in-a-nutanix-cluster-38011

### Create vRouterVM

Install CentOS 7
 4 vCPU, 16 GB mem, 40 GB disk
 vNIC:
  1. bridge: br0, vlan-id 0 (it can be created from webui)
  2. bridge: bro001, vlan-id 0 (this needs to be created by acli command)
  ```
  acli vm.nic_create contrail-controller1 vlan_mode=kTrunked network=__overlay-vm1
  ```

Assign eth0 IP address, and let eth1 unmodified

[install contrail module]


  ```
ansible-playbook -e orchestrator=vcenter -i inventory/ playbooks/configure_instances.yml
ansible-playbook -e orchestrator=vcenter -i inventory/ playbooks/install_contrail.yml
  ```

 - ESXi / vCenter parameters are needed to complete installation

  ```
# cat config/instances.yaml
provider_config:
  bms:
    ssh_pwd: root
    ssh_user: root
    ntpserver: time.google.com
    domainsuffix: local
instances:
  contrail-controller1:
    provider: bms
    ip: x.x.x.x
    esxi_host: y.y.y.y ### not used
    roles:
      config_database:
      config:
      control:
      analytics:
      webui:
      vcenter_plugin:
      vrouter:
      vcenter_manager:
        ESXI_USERNAME: root ### not used
        ESXI_PASSWORD: Dummy123! ### not used
contrail_configuration:
  CLOUD_ORCHESTRATOR: vcenter
  CONTRAIL_CONTAINER_TAG: latest
  CONFIG_NODEMGR__DEFAULTS__minimum_diskGB: 5
  DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 5
  ANALYTICS_STATISTICS_TTL: 2
  JVM_EXTRA_OPTS: "-Xms128m -Xmx4g"
  VCENTER_SERVER: z.z.z.z ### not used
  VCENTER_USERNAME: administrator@vsphere.local ### not used
  VCENTER_PASSWORD: Dummy123! ### not used
  VCENTER_DATACENTER: CN-DC1 ### not used
  VCENTER_DVSWITCH: vm_dvs ### not used
  VCENTER_WSDL_PATH: /usr/src/contrail/contrail-web-core/webroot/js/vim.wsdl ### not used
  VCENTER_AUTH_PROTOCOL: https ### not used
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric
  ```


### patch TVM container

https://github.com/tungstenfabric/tf-vcenter-manager
 - idea is to replace vCenter API to Prism Rest API


### vRouter output


```
 - vif --list is similar to CVM (each tenant vif is associated to eth1, and vlan-id is assigned, vmis in the same vns are still assigned to the same vrfs)

# vif --list
Vrouter Interface Table

Flags: P=Policy, X=Cross Connect, S=Service Chain, Mr=Receive Mirror
       Mt=Transmit Mirror, Tc=Transmit Checksum Offload, L3=Layer 3, L2=Layer 2
       D=DHCP, Vp=Vhost Physical, Pr=Promiscuous, Vnt=Native Vlan Tagged
       Mnp=No MAC Proxy, Dpdk=DPDK PMD Interface, Rfl=Receive Filtering Offload, Mon=Interface is Monitored
       Uuf=Unknown Unicast Flood, Vof=VLAN insert/strip offload, Df=Drop New Flows, L=MAC Learning Enabled
       Proxy=MAC Requests Proxied Always, Er=Etree Root, Mn=Mirror without Vlan Tag, HbsL=HBS Left Intf
       HbsR=HBS Right Intf, Ig=Igmp Trap Enabled

vif0/0      OS: eth0 NH: 4
            Type:Physical HWaddr:50:6b:8d:9c:df:bb IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:TcL3L2VpEr QOS:-1 Ref:4
            RX packets:91040837  bytes:29732460214 errors:46004
            TX packets:106475  bytes:15589023 errors:0
            Drops:46004

vif0/1      OS: vhost0 NH: 5
            Type:Host HWaddr:50:6b:8d:9c:df:bb IPaddr:x.x.x.x
            Vrf:0 Mcast Vrf:65535 Flags:L3DErMn QOS:-1 Ref:8
            RX packets:106374  bytes:15585115 errors:0
            TX packets:91049302  bytes:29732815744 errors:0
            Drops:4

vif0/2      OS: eth1 NH: 12
            Type:Physical HWaddr:50:6b:8d:fe:2e:0e IPaddr:0.0.0.0
            Vrf:0 Mcast Vrf:65535 Flags:TcL3L2PrEr QOS:-1 Ref:6
            RX packets:0  bytes:0 errors:0
            TX packets:106  bytes:4876 errors:0
            Drops:0

vif0/3      OS: pkt0
            Type:Agent HWaddr:00:00:5e:00:01:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3Er QOS:-1 Ref:3
            RX packets:8676  bytes:746560 errors:0
            TX packets:822103  bytes:88304590 errors:0
            Drops:0

vif0/4          6738b223-50a1-30b5-8d5e-b93a1151dc6f Vlan(o/i)(,S): 1/1, 50:6b:8d:92:f2:e7 Bridge Index: 252 Parent:vif0/2 NH: 30
            Type:Virtual(Vlan) HWaddr:50:6b:8d:fe:2e:0e IPaddr:10.0.11.252
            Vrf:2 Mcast Vrf:2 Flags:PL3L2DErMn QOS:-1 Ref:6
            RX packets:0  bytes:0 errors:0
            TX packets:28  bytes:1288 errors:0
            Drops:0

vif0/5          90b15d63-97ba-300a-b520-f85e6f58c951 Vlan(o/i)(,S): 2/2, 50:6b:8d:e3:99:a4 Bridge Index: 156 Parent:vif0/2 NH: 25
            Type:Virtual(Vlan) HWaddr:50:6b:8d:fe:2e:0e IPaddr:10.0.11.251
            Vrf:2 Mcast Vrf:2 Flags:PL3L2DErMn QOS:-1 Ref:6
            RX packets:0  bytes:0 errors:0
            TX packets:25  bytes:1150 errors:0
            ISID: 0 Bmac: 50:6b:8d:e3:99:a4
            Drops:0

vif0/6          931312e4-b0e0-38ba-87d8-53d91696f831 Vlan(o/i)(,S): 3/3, 50:6b:8d:86:53:0c Bridge Index: 804 Parent:vif0/2 NH: 35
            Type:Virtual(Vlan) HWaddr:50:6b:8d:fe:2e:0e IPaddr:10.0.12.252
            Vrf:3 Mcast Vrf:3 Flags:PL3L2DErMn QOS:-1 Ref:6
            RX packets:0  bytes:0 errors:0
            TX packets:28  bytes:1288 errors:0
            ISID: 0 Bmac: 50:6b:8d:86:53:0c
            Drops:0

vif0/7          7852a425-f173-34d7-9998-24fe51179182 Vlan(o/i)(,S): 4/4, 50:6b:8d:8c:07:c5 Bridge Index: 44 Parent:vif0/2 NH: 29
            Type:Virtual(Vlan) HWaddr:50:6b:8d:fe:2e:0e IPaddr:10.0.12.251
            Vrf:3 Mcast Vrf:3 Flags:PL3L2DErMn QOS:-1 Ref:6
            RX packets:0  bytes:0 errors:0
            TX packets:25  bytes:1150 errors:0
            Drops:0

vif0/4350   OS: pkt3
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

vif0/4351   OS: pkt1
            Type:Stats HWaddr:00:00:00:00:00:00 IPaddr:0.0.0.0
            Vrf:65535 Mcast Vrf:65535 Flags:L3L2 QOS:0 Ref:1
            RX packets:0  bytes:0 errors:0
            TX packets:0  bytes:0 errors:0
            Drops:0

#




 - dhcp response is ok (dhcp ack, offer is sent from vRouter)

# ./ist.py vr service
(snip)
DhcpStats
  dhcp_discover: 1
  dhcp_request: 1
  dhcp_inform: 0
  dhcp_decline: 0
  dhcp_other: 0
  dhcp_errors: 0
  offers_sent: 1
  acks_sent: 1
  nacks_sent: 0
  relay_request: 0
  relay_response: 0



 - meta data ip (169.254.0.x) is ok
# ip route
default via x.x.x.1 dev vhost0
169.254.0.1 dev vhost0 proto 109 scope link
169.254.0.4 dev vhost0 proto 109 scope link
169.254.0.5 dev vhost0 proto 109 scope link
169.254.0.6 dev vhost0 proto 109 scope link
169.254.0.7 dev vhost0 proto 109 scope link
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
x.x.x.0/24 dev vhost0 proto kernel scope link src x.x.x.x


- IftReq is ok
# ./ist.py vr intf
+-------+--------------------------------------+--------+-------------------+----------------+---------------+---------+--------------------------------------+
| index | name                                 | active | mac_addr          | ip_addr        | mdata_ip_addr | vm_name | vn_name                             |
+-------+--------------------------------------+--------+-------------------+----------------+---------------+---------+--------------------------------------+
| 0     | eth0                                 | Active | n/a               | n/a            | n/a           | n/a     | n/a                                 |
| 2     | eth1                                 | Active | n/a               | n/a            | n/a           | n/a     | n/a                                 |
| 1     | vhost0                               | Active | 50:6b:8d:9c:df:bb | x.x.x.x        | 169.254.0.1   | n/a     | default-domain:default-project:ip-   |
|       |                                      |        |                   |                |               |         | fabric                              |
| 4     | 6738b223-50a1-30b5-8d5e-b93a1151dc6f | Active | 50:6b:8d:92:f2:e7 | 10.0.11.252    | 169.254.0.4   | vm11-1  | default-domain:vCenter:vn11          |
| 7     | 7852a425-f173-34d7-9998-24fe51179182 | Active | 50:6b:8d:8c:07:c5 | 10.0.12.251    | 169.254.0.7   | vm12-1  | default-domain:vCenter:vn12          |
| 5     | 90b15d63-97ba-300a-b520-f85e6f58c951 | Active | 50:6b:8d:e3:99:a4 | 10.0.11.251    | 169.254.0.5   | vm11    | default-domain:vCenter:vn11          |
| 6     | 931312e4-b0e0-38ba-87d8-53d91696f831 | Active | 50:6b:8d:86:53:0c | 10.0.12.252    | 169.254.0.6   | vm12    | default-domain:vCenter:vn12          |
| 3     | pkt0                                 | Active | n/a               | n/a            | n/a           | n/a     | n/a                                 |
+-------+--------------------------------------+--------+-------------------+----------------+---------------+---------+--------------------------------------+
# ping 169.254.0.4
PING 169.254.0.4 (169.254.0.4) 56(84) bytes of data.
64 bytes from 169.254.0.4: icmp_seq=1 ttl=63 time=3.47 ms
64 bytes from 169.254.0.4: icmp_seq=2 ttl=63 time=2.07 ms
^C
--- 169.254.0.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1004ms
rtt min/avg/max/mdev = 2.079/2.778/3.478/0.701 ms



# ssh cirros@169.254.0.4
The authenticity of host '169.254.0.4 (169.254.0.4)' can't be established.
ECDSA key fingerprint is SHA256:QO9E2ze0zDOixLw9BqEum6KKUSYBzVO35+rMmJyiQn8.
ECDSA key fingerprint is MD5:1b:6c:cc:c6:68:77:a2:ab:66:b9:81:01:24:71:2e:bd.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '169.254.0.4' (ECDSA) to the list of known hosts.
cirros@169.254.0.4's password:
$
$
$
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 50:6b:8d:92:f2:e7 brd ff:ff:ff:ff:ff:ff
2: eth0    inet 10.0.11.252/24 brd 10.0.11.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::526b:8dff:fe92:f2e7/64 scope link \       valid_lft forever preferred_lft forever
$
$
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 50:6b:8d:92:f2:e7 brd ff:ff:ff:ff:ff:ff
$
$



- same subnet ping is working

$ ping 10.0.11.251
PING 10.0.11.251 (10.0.11.251): 56 data bytes
64 bytes from 10.0.11.251: seq=0 ttl=64 time=10.377 ms
64 bytes from 10.0.11.251: seq=1 ttl=64 time=0.972 ms
64 bytes from 10.0.11.251: seq=2 ttl=64 time=1.011 ms
^C
--- 10.0.11.251 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.972/4.120/10.377 ms
$
$
$ arp -n -a
? (10.0.11.251) at 50:6b:8d:e3:99:a4 [ether]  on eth0
? (10.0.11.254) at 50:6b:8d:fe:2e:0e [ether]  on eth0
? (10.0.11.1) at <incomplete>  on eth0
? (10.0.11.253) at 50:6b:8d:fe:2e:0e [ether]  on eth0
$




253, 254 's mac address is from vRouter's eth1:


$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 50:6b:8d:9c:df:bb brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 50:6b:8d:fe:2e:0e brd ff:ff:ff:ff:ff:ff
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:12:29:49:28 brd ff:ff:ff:ff:ff:ff
5: pkt1: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 66:33:40:e2:fd:48 brd 00:00:00:00:00:00
6: pkt3: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void c2:a6:fb:f5:ce:1c brd 00:00:00:00:00:00
7: pkt2: <UP,LOWER_UP> mtu 65535 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/void 42:b3:65:f0:fe:a9 brd 00:00:00:00:00:00
8: vhost0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 50:6b:8d:9c:df:bb brd ff:ff:ff:ff:ff:ff
9: pkt0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 72:ad:f4:4d:d5:89 brd ff:ff:ff:ff:ff:ff



same subnet ping still goes through vRouter (micro-segmentation, same behavior with TVM)
 -> flow 512820: count is increasing


$ flow -l
Flow table(size 161218560, entries 629760)

Entries: Created 171 Added 171 Deleted 332 Changed 332Processed 171 Used Overflow entries 0
(Created Flows/CPU: 49 49 73 0)(oflows 0)

Action:F=Forward, D=Drop N=NAT(S=SNAT, D=DNAT, Ps=SPAT, Pd=DPAT, L=Link Local Port)
 Other:K(nh)=Key_Nexthop, S(nh)=RPF_Nexthop
 Flags:E=Evicted, Ec=Evict Candidate, N=New Flow, M=Modified Dm=Delete Marked
TCP(r=reverse):S=SYN, F=FIN, R=RST, C=HalfClose, E=Established, D=Dead

    Index                Source:Port/Destination:Port                      Proto(V)
-----------------------------------------------------------------------------------
    77264<=>305912       10.0.11.254:39170                                   1 (2)
                         10.0.11.251:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):14,  Stats:0/0,  SPort 60461,
 TTL 0, Sinfo 0.0.0.0)

   255132<=>512820       10.0.11.252:46338                                   1 (2)
                         10.0.11.251:0
(Gen: 1, K(nh):30, Action:F, Flags:, QOS:-1, S(nh):30,  Stats:48/4704,
 SPort 50815, TTL 0, Sinfo 4.0.0.0)

   292028<=>398064       x.x.x.x:49494                                6 (0->2)
                         169.254.0.4:22
(Gen: 1, K(nh):5, Action:N(SD), Flags:, TCP:SSrEEr, QOS:-1, S(nh):10,
 Stats:429/33993,  SPort 60311, TTL 0, Sinfo 0.0.0.0)

   305912<=>77264        10.0.11.251:39170                                   1 (2)
                         10.0.11.254:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):25,  Stats:1/98,  SPort 56475,
 TTL 0, Sinfo 5.0.0.0)

   398064<=>292028       10.0.11.252:22                                      6 (2->0)
                         10.0.11.253:49494
(Gen: 1, K(nh):30, Action:N(SD), Flags:, TCP:SSrEEr, QOS:-1, S(nh):30,
 Stats:294/40078,  SPort 55557, TTL 0, Sinfo 4.0.0.0)

   512820<=>255132       10.0.11.251:46338                                   1 (2)
                         10.0.11.252:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):25,  Stats:48/4704,
 SPort 63370, TTL 0, Sinfo 5.0.0.0)


$ flow -l
Flow table(size 161218560, entries 629760)

Entries: Created 171 Added 171 Deleted 332 Changed 332Processed 171 Used Overflow entries 0
(Created Flows/CPU: 49 49 73 0)(oflows 0)

Action:F=Forward, D=Drop N=NAT(S=SNAT, D=DNAT, Ps=SPAT, Pd=DPAT, L=Link Local Port)
 Other:K(nh)=Key_Nexthop, S(nh)=RPF_Nexthop
 Flags:E=Evicted, Ec=Evict Candidate, N=New Flow, M=Modified Dm=Delete Marked
TCP(r=reverse):S=SYN, F=FIN, R=RST, C=HalfClose, E=Established, D=Dead

    Index                Source:Port/Destination:Port                      Proto(V)
-----------------------------------------------------------------------------------
    77264<=>305912       10.0.11.254:39170                                   1 (2)
                         10.0.11.251:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):14,  Stats:0/0,  SPort 60461,
 TTL 0, Sinfo 0.0.0.0)

   255132<=>512820       10.0.11.252:46338                                   1 (2)
                         10.0.11.251:0
(Gen: 1, K(nh):30, Action:F, Flags:, QOS:-1, S(nh):30,  Stats:77/7546,
 SPort 50815, TTL 0, Sinfo 4.0.0.0)

   292028<=>398064       x.x.x.x:49494                                6 (0->2)
                         169.254.0.4:22
(Gen: 1, K(nh):5, Action:N(SD), Flags:, TCP:SSrEEr, QOS:-1, S(nh):10,
 Stats:458/35501,  SPort 60311, TTL 0, Sinfo 0.0.0.0)

   305912<=>77264        10.0.11.251:39170                                   1 (2)
                         10.0.11.254:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):25,  Stats:1/98,  SPort 56475,
 TTL 0, Sinfo 5.0.0.0)

   398064<=>292028       10.0.11.252:22                                      6 (2->0)
                         10.0.11.253:49494
(Gen: 1, K(nh):30, Action:N(SD), Flags:, TCP:SSrEEr, QOS:-1, S(nh):30,
 Stats:323/45240,  SPort 55557, TTL 0, Sinfo 4.0.0.0)

   512820<=>255132       10.0.11.251:46338                                   1 (2)
                         10.0.11.252:0
(Gen: 1, K(nh):25, Action:F, Flags:, QOS:-1, S(nh):25,  Stats:77/7546,
 SPort 63370, TTL 0, Sinfo 5.0.0.0)


```


## Screenshot

![Dashboard](https://github.com/tnaganawa/ahv-tungstenfabric-integration/blob/main/dashboard.png)
![VmView](https://github.com/tnaganawa/ahv-tungstenfabric-integration/blob/main/vm-view.png)
![NetworkConfig](https://github.com/tnaganawa/ahv-tungstenfabric-integration/blob/main/network-config.png)
