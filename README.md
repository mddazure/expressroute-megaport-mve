# Connecting an ExpressRoute circuit to Megaport MVE Cisco 8000v

Megaport is an ExpressRoute partner in many [locations](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-locations?tabs=america%2Cj-m%2Cus-government-cloud%2Ca-C#global-commercial-azure). The [Megaport Cloud Router (MCR)](https://docs.megaport.com/mcr/) allows ExpressRoute customers to connect leased lines to their onpremise locations, and to connect other Cloud Providers. MCR is easy to set up and operate, it even automatically configures the ExpressRoute Private Peering on both the Megaport and Azure sides, but it does not have a command line interface and does not permit advanced configuration.

For advanced scenario's, [Megaport Virtual Edge (MVE)](https://docs.megaport.com/mve/) provides a platform to run fully configurable Network Virtual Appliances (NVAs) from a variety of vendors. 

This post describes how to connect ExpressRoute to MVE running a Cisco 8000v NVA. 

![image](/images/expressroute-megaport-mve.png)

## Create the Expressroute Circuit 
In the Azure portal, create an  ExpressRoute circuit with Standard Resilliency in a Peering location where Megaport is available.

![image](/images/create-exr-cct.png)

When the cicruit deployment completes, copy the Service key.

![image](/images/s-key.png)

## Create MVE and ExpressRoute connections
Log in to the [Megaport management portal](https://portal.megaport.com/), go to Services and click Create MVE.
- Select Cisco C8000 as the Vendor / Product.
On the next screen:
- Select the Location where the MVE is to be deployed - use the ExpressRoute peering location.
- Select the MVE size.

![image](/images/mve-configure-service.png)

On the following screen:
- Select Autonomous under Appliance Mode.
- Paste a [2048-bit RS SSH public key](https://learn.microsoft.com/en-us/viva/glint/setup/sftp-ssh-key-gen) in the box.
- Under Virtual Interfaces (vNICs), add vNICs as needed. One ExpressRoute circuit requires 2 vNICs, one for each path.
vNIC0 will be used to connect a Megaport Internet VXC for SShh access to the device.

![image](/images/mve-configure-product.png)

On the following screen, give the MVE a name under Finalize Details in the left bar, verify the Summary, and and click Add MVE.

Clicking Create Megaport Internet in the pop up that now appears lets you directly to provision an internet VXC:

- Select the location with the lowest latest latency to the MVE - this will be at the top of the list.

On the next screen:
- Leave the name as proposed or change as needed.
- Set Rate Limit to 20 Mbps (lowest possible, this is for SSH access only).
- Leave A-vNIC set to vNIC-0.
- Leave Preferred A-End VLAN at Untagged.

On the next screen verify the configuration and click Add VXC.

On the main Services page, the MVE and Internet VXC now show with the note "Order pending". 

Click +Connection in the MVE box to add a VXC to the ExpressRoute Circuit.
- Under Choose Destination Type select Cloud.
- Then select Microsoft Azure as the Provider.
- Paste in the circuit's Service Key and select Port for the Primary path.
- Click Next.

![image](/images/vxc-exr-primary.png)

On the next screen:
- Give the connection a name.
- Leave the Rate Limit as proposed, this is set to the bandwdith of the circuit.
- At A-end vNIC, select vNIC-1 (do not leave this at vNIC-0!).
- At Preferred A-End VLAN, turn off Untag and enter a VLAN number. This will be used to set the subinterface in the MVE configuration later.
Scroll down to Azure peering VLAN.
- Leave Configure Azure Peering VLAN turned on.
- Enter the same VLAN ID that will be used in the configuration of the Private Peering on the Azure end.
- Click Next.

Verify the configuration summary and click Add VXC.

![image](/images/megaport-conn-config-pri.png)

Repeat the process to add the Secondary path, terminating on vNIC-2. Enter a different VLAN ID for Preferred A-End VLAN. Enter the same VLAN ID that will be used in the Private Peering under Azure peering VLAN.

![image](/images/megaport-conn-config-sec.png)

When the second ExpressRoute VXC is configured, click Review Order in the right hand bar of the Services screen.

When the validation completes, click Order Now.

This will provision the MVE and the VXC. It will take a few minutes for all services to come up.

In the Azure portal, the Provider Status of the ExpressRoute circuit will change to Provisioned.

## Configure Private Peering
Go back to the ExpressRoute circuit in the Azure portal. The Provisioning Status will now be Provisioned, and the Private Peering can be enabled. Click on Peerings under Settings and then click Azure private.

Enter the Peer ASN and Primary and Secondary subnets. Under VLAN ID enter the **same number configured under Azure Peering VLAN in the Primary and Secondary VXC configurations** in the Megaport portal.

![image](/images/private-peering.png)

## Configure Cisco IOS
Establish an SSH session to the MVE. Use the public ip address from the internet vxc, and the private key that belongs with the public key used when deploying the MVE.

`ssh -i <private-key-file> mveadmin@<public ip>`

Configure interfaces:

```
interface GigabitEthernet2
 no ip address
 no shutdown
 negotiation auto
!
interface GigabitEthernet2.100
 encapsulation dot1Q 100
 ip address 192.168.0.1 255.255.255.252
!
interface GigabitEthernet3
 no ip address
 no shutdown
 negotiation auto
!
interface GigabitEthernet3.101
 encapsulation dot1Q 101
 ip address 192.168.0.5 255.255.255.252
 ```
 Use the Preferred A-end VLAN values set in the primary and secondary VXCs to configure the encapsulation on the subinterfaces. Use the lower address of the /30 subnets configured on the Private Peering.

 The higher IP addresses of the Private Peering should now respond to ping:
 ```
 ping 192.168.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.0.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
If ping does not work there likely is an ARP resolution issue. Run `show arp` and `debug arp` and check the ARP table of the Private Peering.

Configure BGP:
```
router bgp 64000
 bgp log-neighbor-changes
 neighbor 192.168.0.2 remote-as 12076
 neighbor 192.168.0.2 soft-reconfiguration inbound
 neighbor 192.168.0.6 remote-as 12076
 neighbor 192.168.0.6 soft-reconfiguration inbound
 ```
 Verify both neighbor show BGP state = Established:
```
sh ip bgp neighbor 192.168.0.2
BGP neighbor is 192.168.0.2,  remote AS 12076, external link
  BGP version 4, remote router ID 192.168.0.2
  BGP state = Established, up for 1d21h
  ...
```

The basic configuration of ExpressRoute to MVE is now complete.




