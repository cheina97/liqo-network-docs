# IP

The IP resource serves two main purposes. First, it allocates IPs to prevent other components from using them. Second, it maps local IPs to an external CIDR IP, allowing remote adjacent clusters to reach those IPs. For example, if cluster A is peered with cluster B and an IP resource exists on cluster A with the "local IP" set to 10.111.129.179, its status will be updated with an IP from the local external CIDR. This external IP can then be used by adjacent clusters to reach 10.111.129.179.

This mechanism is useful for leaf-to-leaf communication because cluster A is unaware of the pod CIDR used by cluster C. Cluster B exposes cluster C's pods to A using the IP resource.

IP resources are managed by a controller that creates firewall configurations (fwcfg) for the gateways.

### \<tenant-name\>-unknown-source

This IP is the first address within the external CIDR. It is used in the **service-nodeport-routing** firewall configuration (see the relevant section). This IP is used when a packet originates from a NodePort on a leaf cluster and needs to reach a pod in another leaf cluster.

**The related _ipmapping-gw_ firewall configuration is not needed.**
