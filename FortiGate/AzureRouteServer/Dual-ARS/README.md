## Dual Route server architecture

One of the most common architectures in Azure is the hub and spoke design, where workloads deployed in a spoke VNet send traffic through shared network devices that exist in a hub VNet. User Defined Routes (UDR) typically need to be configured in the spoke VNets to steer traffic towards security devices in the hub. However, this requires administrators to manage these routes across many spokes.

Azure Route Server offers a centralized point where Network Virtual Appliances (NVAs) can inject routes that will be programmed for every virtual machine in the spoke, thus eliminating the need for spoke administrators to create and update route tables.

Azure Route Server has dual role:
  - It learns and advertises routes to/from Virtual Network Gateways (VPN and ExpressRoute)
  - It configures learned routes on its VNet, and on directly peered VNets

  This dual functionality often is interesting, but at times it can be detrimental to certain requirements. For example, if the Route Server is deployed in a VNet with an NVA advertising a 0.0.0.0/0 route and an ExpressRoute gateway advertising prefixes from on-premises, it will configure all routes (both the 0.0.0.0/0 from the NVA and the on-premises prefixes) on the virtual machines in its VNet and directly peered VNets. As a consequence, since the on-premises prefixes will be more specific than 0.0.0.0/0, traffic between on-premises and Azure will bypass the NVA in this situation FortiGate VM appliance deployed in Cloud Security Services Hub Vnet. If this isn't desired, there are [articles](https://learn.microsoft.com/en-us/azure/route-server/route-injection-in-spokes) explaining how to disable BGP propagation in the VM subnets and configure UDRs (User Defined Routes) to steer the traffic accordingly.

  However, there's an alternative, more dynamic approach. It's possible using different Azure Route Servers for different functionality: one of them will be responsible for interacting with the Virtual Network Gateways, and the other one for interacting with the Virtual Network routing. The following diagram shows a possible design for this:

 ![Dual-ars](../Dual-ARS/images/msft_dual_ars.png)

In this example from Microsoft Azure Route Server 1 in the hub is used to inject the prefixes from the SDWAN into ExpressRoute. Since the spokes are peered with the hub VNet without the "Use Remote Gateways" and "Allow Gateway Transit" VNet peering options, the spokes won't learn these routes (neither the SDWAN prefixes nor the ExpressRoute prefixes).

To propagate routes to the spokes the NVA-FortiGate uses a second Azure Route Server 2, deployed in a new auxiliary VNet. FortiGate will only propagate a single 0.0.0.0/0 route to this Azure Route Server 2. Since the spokes are peered with this auxiliary VNet with "Use Remote Gateways" and "Allow Gateway Transit" VNet peering options, this 0.0.0.0/0 route will be learned by all the Virtual Machines in the spokes.

The next hop for this 0.0.0.0/0 route will be the FortiGate, so the spokes still need to be peered to the hub VNet. Another important aspect to notice is that the hub VNet needs to be peered to the VNet where the new Azure Route Server 2 is deployed, otherwise it will not be able to create the BGP adjacency.

This design allows automatic injection of routes in a spoke VNets without interference from other routes learned from ExpressRoute, VPN or an SDWAN environment.

