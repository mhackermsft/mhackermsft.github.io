---
title: 'Fine-Grained BGP Control With Azure Route Server Route Maps: A Hands-On Guide for Government Networks'
date: 2026-08-04T16:17:06+00:00
author: Mike Hacker
tags:
- How To
- App Modernization
- Security
categories:
- Networking
summary: A deeply technical walkthrough of Azure Route Server route maps for filtering, summarizing, and reshaping BGP routes across ExpressRoute, VPN, and NVA connections in hybrid government networks.
draft: false
image_prompt: A dramatic close-up photograph of a large brass mechanical mail-sorting machine, with dozens of open pigeonhole channels arranged in a fan, and a single operator's gloved hand redirecting a stack of tagged parcels along different brass chutes. Warm golden-hour light streams across the polished metal, casting long soft shadows and highlighting the intricate routing tracks branching in multiple directions. The composition emphasizes deliberate sorting and channeling of items down chosen paths. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Hybrid connectivity is the backbone of most public sector cloud footprints. A city government running line-of-business systems on-premises while modernizing workloads in Azure, or a state agency stitching together dozens of departmental networks over ExpressRoute and VPN, quickly discovers that the hard part isn't the tunnels - it's the routing. Border Gateway Protocol (BGP) makes hybrid routing dynamic, but dynamic routing without control is how you leak a data center prefix into a partner network or blow past an ExpressRoute route limit.

Azure Route Server has been the managed BGP route reflector at the center of these topologies for years. What has been missing is granular, per-connection control over which routes flow where and how they are shaped. **Route maps for Azure Route Server** fill that gap. This capability remains in public preview as of this writing, so treat it as production-adjacent: excellent for design validation and non-critical paths, but confirm your support posture before you put it in the middle of a mission-critical circuit.

This post is written for the network engineers who actually own the BGP tables. We will cover what route maps do, where they attach, the match-and-action model, and concrete Bicep plus CLI examples.

## What Azure Route Server route maps actually do

Azure Route Server sits between your on-premises networks, network virtual appliances (NVAs), and your Azure virtual networks, exchanging routes over BGP and programming them into the Azure software-defined network. Per the current Microsoft Learn overview (last updated September 2025), it supports up to 16 BGP peers, 4,000 routes per peer, and 10,000 total prefixes.

Route maps let you intercept and reshape routes at the point they enter or leave Route Server. According to the "About route maps for Azure Route Server" documentation on Microsoft Learn, you can apply a route map in two directions on three connection types:

- **Inbound route map** - applied to routes Route Server *receives* from a BGP peering, ExpressRoute gateway, or VPN gateway.
- **Outbound route map** - applied to routes Route Server *sends* to a BGP peering, ExpressRoute gateway, or VPN gateway.

The three attachment points are:

1. BGP peerings between Route Server and your NVAs (SD-WAN, next-gen firewalls, etc.).
2. The ExpressRoute gateway connection in the same virtual network.
3. The VPN gateway connection in the same virtual network.

One important constraint: you can apply **only one route map per direction per connection**. And outbound route maps modify advertisements *only* - they do not influence Route Server's best-path selection, because path selection happens before the outbound map runs. Keep that ordering in mind when you reason about traffic engineering.

## The match-and-action model

A route map is an ordered list of rules. Each rule has **match conditions** and **actions**. This will feel familiar if you have written route-maps on traditional routers, but the Azure implementation has its own vocabulary.

**Match conditions** evaluate a route's prefix, AS-Path, and BGP community:

| Property | Criterion | Behavior |
|----------|-----------|----------|
| Route-prefix | Equals | Matches exactly those prefixes; not more-specifics |
| Route-prefix | Contains | Matches the listed prefixes and everything underneath them |
| Community | Equals | Route must carry all listed communities |
| Community | Contains | Route carries one or more of the listed communities |
| AS-Path | Equals | AS-PATH must contain the ASNs in the listed order |
| AS-Path | Contains | AS-PATH contains one or more listed ASNs, order irrelevant |

If a rule has multiple match conditions, a route must satisfy **all** of them. A route map with no match condition matches every route on that connection.

**Actions** then drop or modify matched routes:

- **Drop** - filter the matched routes out of the advertisement.
- **Route-prefix Replace** - summarize or substitute the matched routes with the prefixes you specify.
- **AS-Path Add** - prepend the listed ASNs (in order) to make a path less preferable.
- **AS-Path Replace** - set the AS-PATH to a specific list, or, with no value, strip it entirely.
- **Community Add / Replace / Remove** - tag, overwrite, or selectively remove BGP communities.

Each rule's **Next step** setting (`Continue` or `Terminate`) decides whether processing proceeds to the next rule or stops. A critical default to internalize: **when no rule matches, the default action is permit, not deny**. If your intent is an allow-list, you must end with an explicit drop rule. Government teams accustomed to default-deny firewall thinking should not assume the same semantics apply here.

## Two scenarios that matter for hybrid government networks

### Scenario 1: Summarizing on-premises routes to stay under ExpressRoute limits

A common pain point: a large agency advertises hundreds of internal /24s from its data center over ExpressRoute. Combined with virtual network prefixes, you risk hitting route-advertisement ceilings - and the Learn documentation notes that when branch-to-branch is enabled, total routes advertised toward an ExpressRoute circuit must not exceed 1,000. An inbound route map on the ExpressRoute gateway connection can aggregate `10.2.1.0/24`, `10.2.2.0/24`, and `10.2.3.0/24` down to a single `10.2.0.0/16`.

One caveat worth flagging loudly: **route summarization strips the BGP Community and AS-PATH attributes from the summarized routes**, for both inbound and outbound. If you depend on community tags for downstream policy, summarize deliberately.

### Scenario 2: Filtering and de-preferring routes from an NVA

Suppose an SD-WAN appliance advertises a broad set of prefixes, but you only want a subset reaching your Azure workloads, and you want a specific path treated as a backup. An inbound route map on the NVA BGP peering can drop the unwanted prefixes, and AS-Path prepending can de-preference a redundant path so it is only used on failover.

Remember the reserved-number rules from the documentation: do **not** prepend private ASNs, and do not use ASNs Azure reserves (public 8074, 8075, 12076; private 65515, 65517-65520). Do not remove Azure's own BGP communities (such as 65517:12001 and the related set). Route maps also support **2-byte ASNs only**.

## Building a route map with Bicep

Route maps are configurable through the Azure portal, and the resource itself is fully expressible as infrastructure-as-code via the `Microsoft.Network/virtualHubs/routeMaps` resource. The schema below is verified against API version `2025-07-01` on Microsoft Learn (page last updated October 2025). Here is an inbound map that summarizes three data center prefixes and drops a management range:

```bicep
param location string = resourceGroup().location
param routeServerHubName string

resource routeMap 'Microsoft.Network/virtualHubs/routeMaps@2025-07-01' = {
  name: '${routeServerHubName}/summarize-onprem-inbound'
  properties: {
    rules: [
      {
        name: 'drop-mgmt-range'
        matchCriteria: [
          {
            matchCondition: 'Contains'
            routePrefix: [ '10.2.250.0/24' ]
          }
        ]
        actions: [
          {
            type: 'Drop'
            parameters: [
              { routePrefix: [ '10.2.250.0/24' ] }
            ]
          }
        ]
        nextStepIfMatched: 'Terminate'
      }
      {
        name: 'summarize-datacenter'
        matchCriteria: [
          {
            matchCondition: 'Contains'
            routePrefix: [ '10.2.0.0/16' ]
          }
        ]
        actions: [
          {
            type: 'Replace'
            parameters: [
              { routePrefix: [ '10.2.0.0/16' ] }
            ]
          }
        ]
        nextStepIfMatched: 'Continue'
      }
    ]
  }
}
```

Note the property names exactly as the ARM schema defines them: `matchCriteria`, `matchCondition`, `nextStepIfMatched`, and action `type` values of `Add`, `Drop`, `Remove`, or `Replace`. Associating the map with a specific connection is done through the `associatedInboundConnections` and `associatedOutboundConnections` properties (arrays of connection resource IDs), or through the portal's connection configuration.

## Verifying behavior from the CLI

Route maps themselves are managed through the portal and ARM/Bicep, but the `az network routeserver` command group is essential for verification. These commands are generally available. After you apply a map, confirm what Route Server is actually learning and advertising:

```bash
# List routes a BGP peer has advertised INTO Route Server (pre/post inbound map)
az network routeserver peering list-learned-routes \
  --name myNvaPeering \
  --routeserver myRouteServer \
  --resource-group myResourceGroup

# List routes Route Server is advertising OUT to a peer (post outbound map)
az network routeserver peering list-advertised-routes \
  --name myNvaPeering \
  --routeserver myRouteServer \
  --resource-group myResourceGroup
```

These two commands are your ground truth. If a summarized prefix is not appearing, or a dropped prefix still shows up, comparing learned versus advertised output tells you exactly where in the pipeline the rule is or isn't firing.

## Operational realities to plan around

From the current documentation, budget for these:

- **First route map triggers a ~30 minute upgrade.** The first time you create a route map on a Route Server, the service undergoes a one-time upgrade of roughly 30 minutes. Schedule accordingly.
- **Route maps incur additional charges** beyond base Route Server pricing.
- **You can modify the default route** (`0.0.0.0/0`) only when it originates from on-premises or an NVA.
- **Route maps cannot modify the virtual network address space** Route Server advertises, and you cannot use route maps to create more-specific routes - only summarization is supported.
- **You cannot apply route maps on the Microsoft Enterprise Edge (MSEE)** for ExpressRoute connections; the map lives on the gateway connection inside your virtual network.
- **IPv6 is not supported** by Azure Route Server.

## A note on Azure US Government and GCC

Many public sector customers operate across both Azure commercial and Azure Government, and staff sit in Microsoft 365 GCC tenants. Route maps are a networking-plane feature of Azure Route Server rather than a Microsoft 365 capability, so GCC tenancy does not gate them. That said, preview features frequently reach the Azure US Government cloud on a later schedule than commercial. Before you standardize a design on route maps in a sovereign region, confirm current regional availability against the Azure Government documentation and validate the exact API version in your target cloud, since ARM API versions can trail commercial. Design your Bicep so the `apiVersion` is a parameter you can adjust per environment.

## Why This Matters for Government

Public sector networks are rarely greenfield. They are decades of accumulated address space, mergers of departmental networks, mandated traffic-inspection paths, and hard external route limits imposed by carriers and circuits. Route maps give network engineers the surgical control they have long had on physical routers, now applied to Azure's managed hybrid fabric without standing up and patching their own route reflectors.

The concrete wins for a city or state IT organization are threefold. First, **route summarization protects you from route-limit outages** - the kind of failure that takes down an ExpressRoute path during a routine advertisement change. Second, **route filtering enforces segmentation intent**: a health department's prefixes should not leak into a public-safety network just because BGP is transitive by default, and an explicit drop rule makes that boundary auditable in code. Third, **AS-Path and community manipulation give you deterministic failover**, so a backup circuit stays a backup until it is genuinely needed - critical when continuity-of-operations plans hinge on predictable path selection.

Because the entire configuration is expressible as Bicep, it also lands in source control, pull requests, and change review - exactly the governance posture auditors expect. That turns routing policy from tribal knowledge on a whiteboard into reviewable, versioned infrastructure.

## Getting started

- Read [About route maps for Azure Route Server](https://learn.microsoft.com/en-us/azure/route-server/route-maps-about) on Microsoft Learn for the full match-and-action reference.
- Review [What is Azure Route Server?](https://learn.microsoft.com/en-us/azure/route-server/overview) for limits and integration patterns (updated September 2025).
- Bookmark the [Microsoft.Network/virtualHubs/routeMaps](https://learn.microsoft.com/en-us/azure/templates/microsoft.network/virtualhubs/routemaps) ARM/Bicep schema (API 2025-07-01).
- Use the [az network routeserver](https://learn.microsoft.com/en-us/cli/azure/network/routeserver) command reference for peering and route verification.

Start in a non-production hub, apply a single summarization rule, and use `list-learned-routes` and `list-advertised-routes` to watch the pipeline behave before you touch a circuit that carries constituent traffic.
