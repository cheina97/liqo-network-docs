# Leaf-to-Leaf Communication

## What is a Leaf Cluster?

Two clusters are considered leaf clusters with respect to a third cluster when both are providers for the same consumer cluster. In this topology, the consumer cluster sits at the center, while the leaf clusters connect to it but not directly to each other. Leaf clusters do not peer directly with one another; instead, their communication is routed through the central (consumer) cluster.

In Liqo, each cluster has visibility only of the network CIDRs of its directly connected (neighbor) clusters. This means that two leaf clusters are not aware of each other's network configuration (e.g., pod CIDR).

As a result, traffic originating from a pod in one leaf cluster cannot directly reach a pod in another leaf cluster, because the source leaf cluster does not know the destination pod's CIDR and how to route the packet. 

The central (consumer) cluster is required to provide the necessary address mapping and routing between the two leaf clusters. This is achieved using the [IP resources](../crds/ip.md).

![Leaf Cluster Topology](../../assets/images/leaftoleaf.excalidraw.png)

## Packets Flow
