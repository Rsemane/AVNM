---
title: "Azure Virtual Network Manager: setting up spoke-to-spoke connectivity"
description: "A beginner-friendly walkthrough of building direct spoke-to-spoke connectivity with AVNM, with real screenshots and the route precedence rules you need to know."
date: 2026-07-06
tags: [azure, networking, avnm, hub-and-spoke, cloud]
layout: post
---

# Azure Virtual Network Manager: setting up spoke-to-spoke connectivity

When an organisation grows in Azure, it rarely keeps a single virtual network. Each team, environment or application usually ends up with its own VNet, and before long you have dozens of networks that need to talk to each other.

The traditional answer is the hub and spoke model. One central hub VNet hosts the shared services, and every application VNet, a spoke, is peered to that hub. It's clean and secure, but it comes with a well known limitation: spokes can't talk to each other directly. Their traffic has to transit through the hub, or you need a dedicated VNet peering between every pair of spokes that has to communicate. That second option doesn't scale well. Five spokes fully meshed already means 10 peerings, and 20 spokes means 190.

This is the problem Azure Virtual Network Manager, or AVNM, is built to solve. You group your spokes into a network group, apply a connectivity configuration with direct connectivity enabled, and AVNM interconnects the members as a Connected Group. From then on, adding a new VNet to the group is enough for it to reach all the others. Nothing to peer manually, nothing to forget.

This post walks through the whole process step by step, with a screenshot for each stage of a real deployment: creating the Network Manager instance, building the network group, setting up the connectivity configuration, deploying it, and finally checking that everything actually works with a traceroute and a look at the effective routes.

You only need basic Azure knowledge for this one, meaning you should already know what a VNet, a subnet and a peering are. Every AVNM specific term is explained the first time it comes up.

## Key concepts in plain words

| Term | What it means |
|------|---------------|
| VNet peering | A manual, point to point link between two VNets, over the Microsoft backbone. |
| Hub and spoke | A topology where all spokes are peered to a central hub hosting shared services. |
| AVNM | Azure Virtual Network Manager, a central service that applies connectivity, security admin and routing configurations to many VNets at once. |
| Management scope | The subscriptions or management groups an AVNM instance is allowed to manage. |
| Network group | A logical container of VNets. Members are added manually, static, or automatically via Azure Policy, dynamic. |
| Connectivity configuration | The rule attached to a network group: hub and spoke or mesh topology, with optional direct connectivity between members. |
| Connected Group | The result of enabling direct connectivity. An implicit, managed mesh between group members, created without any individual peering objects. |

A Connected Group is not peering created for you automatically. You won't see any peering objects between the spokes in the portal, the validation section further down proves that, yet the VNets can still reach each other. A single connected group can hold up to 250 VNets.

## Lab architecture

The environment used for these screenshots is kept deliberately small so the mechanics stay easy to follow: one hub, hub-vnet, two spokes, spoke1-vnet and spoke2-vnet, and one test VM sitting in each spoke. The AVNM instance avnm-core-prod manages them through the network group ng-app-spokes, using the connectivity configuration conn-hub-and-spoke.

![AVNM landing page](images/AVNM_lab_architecture.jpg)

A few things worth noting. The solid lines are the hub to spoke connectivity, created and maintained by the AVNM configuration. The dashed line is the direct spoke to spoke path created by the Connected Group, so traffic between vm1 and vm2 now takes one direct hop instead of going through the hub. And AVNM itself only sits in the management plane. It pushes the configuration down to the network, but it never actually carries any of your traffic.

## Step by step in the Azure portal

### 1. Open Azure Virtual Network Manager

In the portal, search for Virtual network manager. The Get started page gives a quick summary of what the service does and is also where the creation wizard lives.

![AVNM landing page](images/avnm-01-get-started.png)

### 2. Create the Network Manager instance

Click Create and fill in the Basics tab: pick a subscription, a new resource group named rg-avnm, an instance name of avnm-core-prod, and a region, in this case West Europe. That region only hosts the manager itself. It can still manage VNets located anywhere.

![Create network manager, Basics](images/avnm-02-create-basics.png)

The Management scope tab defines which subscriptions or management groups this instance is allowed to manage.

![Add scopes panel](images/avnm-03-scope-add.png)

![Scope added](images/avnm-04-scope-added.png)

On the review screen you can confirm the Connectivity feature is enabled, since that's what allows connected groups to exist at all. Once that looks right, create the instance.

![Review and create](images/avnm-05-review-create.png)

![Deployment complete](images/avnm-06-deployment-complete.png)

### 3. Create a network group and add the spokes

Inside avnm-core-prod, open Network groups and create ng-app-spokes. At this point it's just an empty container.

![Create network group](images/avnm-07-network-group.png)

Now add the spokes as members. Here we add them manually, as static members, by ticking spoke1-vnet and spoke2-vnet. Notice the hub is not part of this group. Only the spokes belong to it.

![Manually add members](images/avnm-08-add-members.png)

In production it's usually better to prefer dynamic membership: define an Azure Policy rule, for example name contains spoke and tag env equals prod. Any future VNet matching that rule joins the group automatically, and therefore the connected group too.

### 4. Create the connectivity configuration

Open Configurations and create a connectivity configuration named conn-hub-and-spoke. Pick ng-app-spokes as the target network group, and this is the important part, tick Direct connectivity within network group. That single checkbox is what actually creates the Connected Group between the spokes. The option Connect network group to a hub is what links every member to hub-vnet.

![Connectivity configuration, Basics](images/avnm-09-connectivity-basics.png)

The Advanced tab offers two useful safeguards: preventing overlapping address spaces, and allowing high scale private endpoints.

![Connectivity configuration, Advanced](images/avnm-10-connectivity-advanced.png)

The Preview tab draws the resulting topology so you can double check it before anything is actually created.

![Topology preview](images/avnm-11-topology-preview.png)

### 5. Deploy the configuration

Creating a configuration doesn't change anything yet. AVNM works on a goal state model, so nothing takes effect until you deploy. Open Deployments, still empty at this stage, and click Deploy configurations.

![Deployments blade](images/avnm-12-deployments-blade.png)

Select the connectivity configuration along with the target region or regions where it should apply.

![Goal state](images/avnm-13-deploy-goal-state.png)

![Review and deploy](images/avnm-14-review-deploy.png)

A few moments later the deployment shows as succeeded for westeurope.

![Deployed](images/avnm-15-deployed.png)

## Validating the result

Three checks are enough to prove the connected group is actually doing its job.

### The spokes still have no peering between them

The three VNets show up as usual under Virtual networks.

![VNet list](images/avnm-18-vnet-list.png)

Now open the Peerings blade of spoke1-vnet. There's no peering towards spoke2-vnet. This is the proof that a Connected Group is not peering created automatically behind the scenes. The connectivity exists without any peering object at all.

![spoke1-vnet peerings](images/avnm-17-spoke1-peerings.png)

### Traffic flows directly between the spokes

From vm1, in spoke1-vnet, a traceroute to vm2 at 10.2.0.4, in spoke2-vnet, completes in a single hop. That single hop means the packet went straight from one spoke to the other without passing through the hub.

![traceroute vm1 to vm2](images/avnm-16-traceroute.png)

### Reading the effective routes

The clearest proof sits on the network interface of the VM itself. Open the VM, then Network settings, then the NIC, then Effective routes.

![Effective routes blade](images/avnm-19-effective-routes.png)

## The system route hierarchy, and who wins

Here's the subtle part. Both VNet peering and Connected Groups are categorised under the hood as default system routes. They are, however, not ranked equally. An explicit, manual, point to point VNet peering outranks an automated AVNM Connected Group.

When several routes advertise the exact same prefix, Azure evaluates them in this order of precedence:

| Priority | Route source | Comment |
|:--------:|--------------|---------|
| 1 | User Defined Routes, UDR | Highest priority, always wins |
| 2 | BGP routes, ExpressRoute or VPN | Learned from on premises |
| 3 | VNet peering, or Global VNet peering | Manual point to point links |
| 4 | AVNM Connected Group | Managed mesh, beaten by peering |
| 5 | VirtualNetwork, local subnets | Lowest priority |

Before any peering exists, the route towards spoke2 shows up with the source ConnectedGroup, sitting next to Default and User routes. As soon as a manual VNet peering is created between spoke1 and spoke2, that ConnectedGroup route disappears from the effective routes and is replaced by a VNetPeering route for the same prefix. Nothing about the connected group itself changed, it simply stopped being the route in use, which is exactly what the precedence table predicts.

![Effective routes table with ConnectedGroup source](images/avnm-20-effective-routes-table.png)

### What this means in practice

Coexistence is safe. If two spokes are both peered directly and members of the same connected group, traffic uses the peering, and the connected group route simply sits behind it without causing any conflict.

Migration is easy. You can deploy the connected group first, verify that it works, and only then delete the old spoke-to-spoke peerings one by one. Traffic falls back automatically from priority 3 to priority 4, with no outage in between.

UDRs always override everything else. If a route table forces 10.2.0.0/16 towards a hub firewall, neither the peering nor the connected group will be used, no matter how they're configured.

The same check is available from the CLI:

```bash
az network nic show-effective-route-table \
  --name vm1623 --resource-group rg-spokes --output table
```

## Best practices and limitations

Keep the hub for north-south traffic such as internet and on-premises access, and reserve direct connectivity for east-west traffic only when latency requirements leave you no other choice.

Prefer dynamic network groups driven by Azure Policy so new spokes join the connected group on their own.

Address spaces must not overlap. Turn on the prevent overlapping address spaces guard found in the Advanced tab.

One connected group can hold up to 250 VNets, and a single VNet can belong to at most two connected groups.

If spoke-to-spoke traffic needs to go through a firewall for inspection, route it via the hub with UDRs. Remember that UDRs always take priority over everything else.

AVNM is billed per managed subscription, so it's worth checking current pricing before scoping it too broadly.

## References

- [Azure Virtual Network Manager overview](https://learn.microsoft.com/azure/virtual-network-manager/overview)
- [Connectivity configurations and connected groups](https://learn.microsoft.com/azure/virtual-network-manager/concept-connectivity-configuration)
- [Virtual network traffic routing and system route precedence](https://learn.microsoft.com/azure/virtual-network/virtual-networks-udr-overview)

---

*Written after deploying AVNM connected groups in a lab environment. All screenshots are from the actual deployment. Questions or corrections are welcome, feel free to open an issue on this repo.*
