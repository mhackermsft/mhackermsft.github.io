---
title: 'Running Dual-Stack IPv6 Through Azure VPN Gateway: A Government Transition Guide'
date: 2026-07-21T18:34:04+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Security
categories:
- Azure
- Government
summary: A hands-on look at dual-stack IPv6 over Azure VPN Gateway, the SKU and subnet requirements, and CLI/Bicep patterns that help public sector teams respond to the federal IPv6 mandate.
draft: false
image_prompt: Azure VPN Gateway dual-stack IPv6 architecture for state and local government, showing an Azure hub VNet, GatewaySubnet, IPv4 VPN outer tunnel, IPv6 inner traffic, and connected government datacenter.
image: cover.png
audio: audio.mp3
---

The federal push toward IPv6 is no longer a distant planning exercise. Office of Management and Budget memo M-21-07, "Completing the Transition to Internet Protocol Version 6 (IPv6)," published November 19, 2020, set aggressive federal targets: 20% of federal IP-enabled assets on IPv6-only by the end of FY2023, 50% by FY2024, and 80% by FY2025. While the memo binds federal agencies directly, it is also a practical reference point for state and local governments because grant conditions, interagency data exchanges, and vendor procurement language often reflect the same direction.

For a city, county, or state IT team standing up hybrid connectivity to Azure, the practical question is concrete: can IPv6 traffic traverse VPN tunnels, or does IPv6 stop at the cloud edge? This post walks through the current state of dual-stack IPv6 with Azure networking, what Azure VPN Gateway supports, the SKU and subnet rules you must respect, and repeatable Azure CLI and Bicep patterns to build the foundation.

> **Availability note up front.** IPv6 for Azure Virtual Network is generally available for dual-stack VNets, subnets, load balancers, NSGs, and related capabilities. Azure VPN Gateway IPv6 support is documented for dual-stack deployments and is listed on the VPN Gateway What's new page as an IPv6 Preview item. Treat the VPN Gateway IPv6 tunnel scenario as preview: validate it in a non-production subscription, confirm regional and Azure Government availability, and check the current VPN Gateway documentation before committing production traffic.

## The dual-stack foundation: what is GA today

Azure's [IPv6 for Azure Virtual Network](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/ipv6-overview) is a full-featured capability for hosting applications with both IPv4 and IPv6 connectivity. You define your own IPv6 address space, run dual-stack VNets and subnets, apply IPv6 rules in Network Security Groups, and front workloads with Standard IPv6 public and internal load balancers.

A few hard rules shape every design:

- **IPv6 subnets must be exactly /64.** The Azure IPv6 overview is explicit: IPv6 subnets must be /64, which also keeps designs compatible with on-premises routers that accept only /64 IPv6 routes.
- **No IPv6-only VMs.** Each NIC must include at least one IPv4 IP configuration. Dual-stack is the supported model for VMs and VM scale sets.
- **Some networking services are still IPv4-only.** The current IPv6 overview lists Azure Virtual WAN, Azure Route Server, and Azure Firewall as not supporting IPv6 traffic. Azure Firewall can operate in a dual-stack VNet using IPv4, but the firewall subnet must be IPv4-only.

That last point matters for transition planning. If IPv6 must traverse a site-to-site or VNet-to-VNet VPN path, use the Azure VPN Gateway dual-stack preview path documented for VPN Gateway rather than designing the IPv6 path around Virtual WAN.

## Choosing a gateway SKU

The Azure VPN Gateway IPv6 configuration documentation lists IPv6 dual-stack support for **VpnGw1AZ through VpnGw5AZ**. For a new government hub, choose a route-based, zone-capable gateway SKU and avoid legacy gateway designs.

| SKU | Generation | Aggregate throughput benchmark | BGP | Zone-redundant support |
|-----|------------|--------------------------------|-----|------------------------|
| VpnGw2AZ | Gen2 | 1.25 Gbps | Yes | Yes |
| VpnGw3AZ | Gen2 | 2.5 Gbps | Yes | Yes |
| VpnGw4AZ | Gen2 | 5 Gbps | Yes | Yes |
| VpnGw5AZ | Gen2 | 10 Gbps | Yes | Yes |

VpnGw1AZ is also listed as an IPv6-capable AZ SKU, but it is a Generation1 gateway with a lower throughput benchmark. For many government hub deployments that need BGP and availability-zone resilience, VpnGw2AZ or VpnGw3AZ is a more practical starting point. Size up only if throughput testing and tunnel count justify it.

Two lifecycle items from the VPN Gateway documentation matter for procurement and migration planning:

- **The Basic VPN Gateway SKU is not retiring, but it is not an IPv6 dual-stack SKU.** The IPv6 dual-stack support list starts at VpnGw1AZ. Also separate the gateway SKU from Basic public IP retirement work: older gateways that still show Basic public IP references need the documented migration or removal path.
- **Legacy and non-AZ gateway SKUs should not be used for new work.** The VPN Gateway What's new page and SKU consolidation documentation list retirement and migration timelines for legacy Standard/High Performance SKUs and non-AZ VpnGw1-5 SKUs. New IPv6 work should use AZ gateway SKUs.

Azure VPN Gateway IPv6 is for inner traffic in the VPN tunnel. The documentation states that IPv6 outer tunnel support is not currently available, so the gateway public IP remains IPv4 in the examples below.

## Building the dual-stack VNet with the Azure CLI

Start with the foundation: a dual-stack VNet, a /64 workload subnet, and a dedicated dual-stack GatewaySubnet. The gateway subnet must be named exactly `GatewaySubnet`, and for the VPN Gateway IPv6 scenario it must include both IPv4 and IPv6 address ranges.

```bash
# Resource group
az group create --name rg-net-ipv6 --location eastus2

# Dual-stack VNet: IPv4 + IPv6 address space
az network vnet create --resource-group rg-net-ipv6 --name vnet-hub --location eastus2 --address-prefixes 10.20.0.0/16 fd00:db8:20::/48

# Workload subnet: IPv6 subnet must be exactly /64
az network vnet subnet create --resource-group rg-net-ipv6 --vnet-name vnet-hub --name snet-workload --address-prefixes 10.20.1.0/24 fd00:db8:20:1::/64

# GatewaySubnet: name is required, and dual-stack VPN Gateway needs IPv4 + IPv6 ranges
az network vnet subnet create --resource-group rg-net-ipv6 --vnet-name vnet-hub --name GatewaySubnet --address-prefixes 10.20.255.0/27 fd00:db8:20:ffff::/64
```

Using a Unique Local Address range such as `fd00::/8` keeps the lab private. In production, substitute the IPv6 prefix assigned to your organization or the prefix your network architecture requires so the address plan integrates cleanly with on-premises routing.

Next, create the IPv4 public IP for the gateway and the route-based VPN gateway. Set the gateway generation explicitly when you want the Generation2 throughput profile for VpnGw2AZ and above.

```bash
az network public-ip create --resource-group rg-net-ipv6 --name pip-vpngw --sku Standard --allocation-method Static --version IPv4 --zone 1 2 3

az network vnet-gateway create --resource-group rg-net-ipv6 --name vpngw-hub --location eastus2 --public-ip-addresses pip-vpngw --vnet vnet-hub --gateway-type Vpn --vpn-type RouteBased --sku VpnGw2AZ --vpn-gateway-generation Generation2 --no-wait
```

Gateway provisioning commonly takes 45 minutes or more, so `--no-wait` is useful in deployment automation. Because IPv6 for VPN Gateway is preview, confirm the current portal, Azure CLI, or PowerShell steps in the VPN Gateway IPv6 configuration documentation for your target region before you automate production changes.

## The same foundation in Bicep

Infrastructure-as-code is where government teams get repeatability and auditability. The following Bicep creates the dual-stack VNet, the /64 workload subnet, the dual-stack GatewaySubnet, and the IPv4 Standard public IP used by the VPN gateway.

```bicep
param location string = resourceGroup().location

resource vnet 'Microsoft.Network/virtualNetworks@2025-07-01' = {
  name: 'vnet-hub'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.20.0.0/16'
        'fd00:db8:20::/48'
      ]
    }
    subnets: [
      {
        name: 'snet-workload'
        properties: {
          addressPrefixes: [
            '10.20.1.0/24'
            'fd00:db8:20:1::/64'
          ]
        }
      }
      {
        name: 'GatewaySubnet'
        properties: {
          addressPrefixes: [
            '10.20.255.0/27'
            'fd00:db8:20:ffff::/64'
          ]
        }
      }
    ]
  }
}

resource pip 'Microsoft.Network/publicIPAddresses@2025-07-01' = {
  name: 'pip-vpngw'
  location: location
  sku: {
    name: 'Standard'
  }
  zones: [
    '1'
    '2'
    '3'
  ]
  properties: {
    publicIPAddressVersion: 'IPv4'
    publicIPAllocationMethod: 'Static'
  }
}
```

Note the use of `addressPrefixes` on the subnets to carry both IPv4 and IPv6 prefixes. Keeping the /64 boundary in code, including on `GatewaySubnet`, prevents one of the most common IPv6 deployment errors: a subnet sized anything other than /64.

## Architecture patterns for the transition

**1. Dual-stack hub-and-spoke.** Run a dual-stack hub VNet with the VPN gateway, and peer dual-stack spokes. VNet peering carries both IPv4 and IPv6 endpoints, and Azure supports peering a dual-stack VNet with an IPv4-only VNet during migration. That lets you convert workloads incrementally instead of forcing a single cutover.

**2. Keep the firewall lane IPv4-first unless you choose an IPv6-capable NVA.** Azure Firewall does not currently support IPv6 traffic. Place Azure Firewall in an IPv4-only subnet within the dual-stack VNet, and design IPv6 inspection deliberately with NSG-based controls or a third-party network virtual appliance that supports IPv6.

**3. Choose VPN Gateway when IPv6 must cross the VPN path.** Virtual WAN currently supports IPv4 traffic only. For the documented Azure VPN Gateway dual-stack preview scenario, use a route-based VPN gateway and document why the IPv6 path is not built on Virtual WAN.

**4. Secure IPv6 explicitly.** IPv6 Internet connectivity is established only when you request it, and Azure DDoS protections extend to Internet-facing IPv6 public IPs. Build matching IPv6 NSG rules alongside IPv4 rules. Remember that ICMPv6 is not currently supported in NSGs, which affects reachability testing.

**5. Respect protocol limits.** The VPN Gateway IPv6 documentation states that point-to-site IPv6 is supported with IKEv2 and OpenVPN, not SSTP. Site-to-site IPv6 is not supported with IKEv1. Plan device configuration and testing around those limits.

## Why This Matters for Government

State and local agencies rarely get to transition on their own timeline. IPv6 requirements can arrive through federal grant language, data-sharing agreements with agencies already bound by M-21-07, and the address-exhaustion reality of IoT-heavy public services such as transit sensors, utility metering, and public safety devices. Being able to demonstrate an IPv6-capable network architecture is becoming a practical prerequisite for interagency work.

The compliance angle is also important. Even when the workload is not directly federal, network teams may need architecture evidence for StateRAMP, FedRAMP-aligned controls, CJIS segmentation, HIPAA workloads, or IRS Publication 1075 environments. A dual-stack design expressed in Bicep gives reviewers a concrete artifact: address spaces, subnet boundaries, gateway SKUs, NSG posture, and migration sequencing are visible in code rather than scattered across tickets and diagrams.

The sequencing is pragmatic. The dual-stack VNet foundation is generally available, so you can build and validate IPv6 addressing, subnetting, NSG posture, and infrastructure-as-code today. The VPN Gateway IPv6 tunnel scenario is preview, so pilot it in a controlled subscription while the production foundation remains sound. Dual-stack lets IPv4 continue to flow while you add IPv6 alongside it.

For budget owners, the message is that IPv6 readiness does not require ripping out existing connectivity. You can size a modern zone-capable gateway, express the design in Bicep for auditability, and expand IPv6 coverage spoke by spoke as workloads are ready. That is a defensible, incremental answer to an external mandate rather than a forklift migration.

## Next steps

- Review [IPv6 for Azure Virtual Network](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/ipv6-overview) for the full list of GA capabilities and current limitations.
- Check [Configure IPv6 for VPN Gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/ipv6-configuration) for the current preview configuration steps and protocol limitations.
- Check the [VPN Gateway What's new page](https://learn.microsoft.com/en-us/azure/vpn-gateway/whats-new) for IPv6 preview status, SKU retirement timelines, and related release notes.
- Confirm gateway sizing against [About Azure VPN Gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways).
- Read OMB [M-21-07](https://www.whitehouse.gov/wp-content/uploads/2020/11/M-21-07.pdf) to map federal milestones against your own roadmap.

Start with the dual-stack foundation, keep every /64 boundary in code, and let IPv6 grow alongside your existing IPv4 network rather than replacing it.
