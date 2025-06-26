# Main Documentation

Welcome to the main section of the Liqo Network Documentation.

Below you will find links to the available topics in this section:

- [Basic](basic.md): This page is reserved for foundational concepts and general information about the Liqo networking model. (Currently empty)
- [Leaf-to-Leaf](leaftoleaf.md): This section will cover the details of leaf-to-leaf networking, including how network traffic is routed between leaf nodes. (Currently empty)
- [NodePort](nodeport.md): Here you will find information about NodePort services and how they are managed within the Liqo network. (Currently empty)

If you want details about each CRDs, you can check:

- [Firewall Configuration](../crds/firewall.md): This page provides detailed information about each firewall configuration resource.
- [Routing Configuration](../crds/routing.md): This page provides detailed information about each routing configuration resource.
- [IP](../crds/ip.md): This page provides detailed information about IP configuration resource and how they work.
- [InternalNode](../crds/internalnode.md): This page provides detailed information about the InternalNode resource and its role in the Liqo network.

## Debug commands

- `tcpdump -tnl -i any \<protocol\>`: capture all traffic for a specific protocol (e.g., `tcp`, `udp`, `icmp`, etc.)
- `tcpdump -tnl -i any tcp port 8080`: capture all TCP traffic on port 8080
- `tcpdump -tnl -i any tcp dst port 8080`: capture traffic destined to port 8080
- `conntrack -D`: delete all the conntrack entries
- `conntrack -L`: list all the conntrack entries
- `nft list ruleset`: list all the rules in the nftables
