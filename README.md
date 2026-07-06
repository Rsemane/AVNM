# Azure networking notes

This repository holds the write-ups I publish on my GitHub Pages site about Azure networking and infrastructure work.

## Latest post

**[Azure Virtual Network Manager: setting up spoke-to-spoke connectivity](avnm-spoke-to-spoke-connectivity.md)**

A beginner-friendly, screenshot-by-screenshot walkthrough of building direct spoke-to-spoke connectivity with AVNM Connected Groups, including why the classic hub and spoke model does not let spokes talk to each other, how to set up the network group and connectivity configuration, and how to prove the connectivity actually works with a traceroute and the effective routes. It also covers the system route hierarchy, meaning what happens when a manual VNet peering and an AVNM Connected Group both exist for the same destination.

## Further reading

If you want to go one level deeper before reading the post above, this article is one of the clearest explanations of what is actually happening under the hood in Azure networking:

- [Azure Virtual Networks Do Not Exist, by Aidan Finn](https://aidanfinn.com/?p=23909)

The short version of his point: a VNet is not a physical network with wires and gateways, it is a mapping that tells the Azure fabric which network interfaces are allowed to reach each other. Every packet travels directly from the source NIC to the destination NIC over the physical network, and peering does not create a cable, it just adds new entries to that mapping. That is exactly why an AVNM Connected Group can make two spokes reach each other without any peering object ever being created between them: there was never a wire to draw in the first place, only routing intent. Reading his article first makes the spoke-to-spoke behaviour described in my post much less mysterious.
