# IP

### Leaf-to-leaf communication

The IP resource is used for 2 purposes. The first one is to allocate IPs to avoid other components use it. Secondly, it is used to map local IPs on an external-cidr IP. This means that remote adjacent clusters can use it to reach that IP. For example if cluster A is peered with cluster B, and an IP resource is present on cluster A with "local IP" equal to 10.111.129.179, its status will be fulfilled with an IP coming from the local external-cidr which can be used to reach 10.111.129.179 from adjacent clusters.

This is useful in leaf-to-leaf communication because cluster A is not aware of the pod-cidr used by cluster C. So cluster B exposes cluster C's pods to A using the IP resource.

The IP resurces are reconciled by a controller which creates fwcfg for the gateways.

### \<tenant-name\>-unknown-source

This IP is the first one inside the external CIDR. It is used inside **service-nodeport-routing** fwcfg (check it). This IP is used when a packets comes from a nodeport on a leaf cluster and need to reach a pod in another leaf one,.

**The related *ipmapping-gw* fwcfg is useless**.

# Firewall Configuration

## Labels

TODO

## Before Peering

### service-nodeport-routing

The biggest problem with nodeport traffic is that there is no a standard way to recognize the traffic origin from the src ip. Let's imagine a packet reach a node on the right port, then it should be src-natted or not (and there is not a standard ip used, it depends from the CNI). To solve this issue, Liqo always uses the first IP of the external-cidr to src-nat the traffic received on a nodeport.

This fwcfg does not apply the src-nat cited before. It contains a **ctmark** rule which create a conntrack with a **mark** (a progressive number different from each node). Then another prerouting chain take all the traffic coming from other clusters with the first ext-cidr ip as dst ip and uses the **conntrack** to apply the **mark**. Then a policy routing rule uses this mark to forward packets to the original node. This is necessary because when the nodeport traffic return from the remote cluster has always the same dst ip. **Following the same path in both directions** is a best practice since some CNIs block traffic which use a different path on the way back.

### \<node-name\>-gw-masquerade-bypass    

Sometimes CNIs masquerade the traffic from a pod to a node (one that is not running the pod) using the starting node (IP). For example if a pod has IP 10.0.0.8 and it is scheduled on a node with internal IP 192.168.0.5, if we try to ping another node, this one will receive packets with 192.168.0.5 as src IP.

This behaviour has been observed in these scenarios:

- Azure CNI
- Calico, when pod masquerade is enabled (Crownlabs)
- KindNet

This can be an issue because Geneve does not support NAT (and changes of IPs) in the middle of the path between the 2 hosts. This means that when a Geneve tunnel need to be established (like between gw-pods and nodes) the 2 hosts should be able to "ping" the other one using its IP (the one used by the network interface).

This fwcfg solves this issue using the same approach described in the **\<tenant-name\>-masquerade-bypass** fwcfg. The double SNAT trick is used also here in order to prevent the masquerade. Every time a gw-pod is scheduled on a node, a rule to prevent masquerade is added only on that specific node (with label networking.liqo.io/firewall-unique). It match only traffic with the gw IP as src IP and the Geneve port as dst port. 

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: FirewallConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/firewall-category: fabric
    networking.liqo.io/firewall-subcategory: single-node
    networking.liqo.io/firewall-unique: cheina-cluster1-worker2
    networking.liqo.io/gateway-masquerade-bypass: "true"
  name: cheina-cluster1-worker2-gw-masquerade-bypass
  namespace: liqo
spec:
  table:
    chains:
    - hook: postrouting
      name: pre-postrouting
      policy: accept
      priority: 99
      rules:
        natRules:
        - match:
          - ip:
              position: src
              value: 10.127.65.33
            op: eq
            port:
              position: dst
              value: "6091"
            proto:
              value: udp
          name: gw-cheina-cluster2-66bc45dd78-75d9j
          natType: snat
          targetRef:
            kind: Pod
            name: gw-cheina-cluster2-66bc45dd78-75d9j
            namespace: liqo-tenant-cheina-cluster2
          to: 10.127.65.33
      type: nat
    family: IPV4
    name: cheina-cluster1-worker2-gw-masquerade-bypass
```

## After Peering

### \<tenant-name\>-masquerade-bypass (Node)

This fwcfg contains several rules with different purposes.

Geneve needs that 2 hosts can communicate directly, without ip changings in the middle (e.g. no nat allowed). Sometimes CNIs apply a SNAT (applying the node primary IP) when traffic starts from pods and try to reach a node. This rules allows to nullify this SNAT applying a useless SNAT. For example if a packet uses 10.0.0.1 as src IP, applying a SNAT which enforce 10.0.0.1 nullify every next SNAT rules. Same for DNAT rules. 

```yaml
- match:
    - ip:
        position: dst
        value: 10.71.0.0/18
      op: eq
    - ip:
        position: src
        value: 10.127.64.0/18
      op: eq
  name: podcidr-cheina-cluster2
  natType: snat
  to: 10.127.64.0/18

```

The next rules enforce the presence of the first ext-cidr IP in packets received by nodeport services. Check the  **service-nodeport-routing** fwcfg.

```yaml
- match:
	- ip:
      	position: dst
        value: 10.71.0.0/18
      op: eq
    - ip:
      	position: src
        value: 10.127.64.0/18
      op: neq
  name: service-nodeport-cheina-cluster2
  natType: snat
  to: 10.70.0.0
```

The other rules apply the same concept but for ext-cidr

**This rule is the reason why nodeports do not work with cilium without kube proxy. With no-kubeproxy cilium uses ebpf to manage firewalling rules, so this fwcfg is bypassed. A solution for the future should be to move this rule inside the Gateway**

#### Full-Masquerade

When the flag **networking.fabric.config.fullMasquerade** is **true**, this fwcfg changes. In particular the **service-nodeport-\<cheina-cluster2>** rule becomes the only one still present, and it's match rules does not includes the check on the src IP of the packets.

```yaml
- match:
	- ip:
      	position: dst
        value: 10.71.0.0/18
      op: eq
  name: service-nodeport-cheina-cluster2
  natType: snat
  to: 10.70.0.0
```

This rule take all the traffic flowing to the remote cluster and force the **unknown IP** as src IP.  It means that the remote traffic will receive all the traffic coming from its peered cluster the first external cidr IP as src IP. 

That's useful when the cluster nodeports uses a PodCIDR IP to masquerade the incoming traffic.

### \<tenant-name\>-remap-podcidr (Gateway)

This rule manage the cidr remapping in case 2 clusters have the same pod-cidr. It contains 2 rules.

Before continue, let's recap how this works:

Let's imagine we have 2 clusters named cluster A and B. they both have the same pod-cidr (e.g. 10.1.0.0/16). Every cluster can remap the cidr of the adiacent ones even with the same pod-cidr. So cluster A can decide autonomously a new cidr to identify the cluster B pods. When the cluster A wants to send traffic to cluster B, it will point to the new remapped cidr. This rule purpose is to translate the "fake" destination IP with the real one. Please notice that this rule ignore traffic coming from eth0, since the traffic from the pods will be received on geneve interfaces (liqo-xxx).

```yaml
- match:
    - ip:
        position: dst
        value: 10.71.0.0/18
      op: eq
    - dev:
        position: in
        value: eth0
      op: neq
    - dev:
        position: in
        value: liqo-tunnel
      op: neq
  name: 17b97d17-aa77-4494-bf9c-d307600f37af
  natType: dnat
  to: 10.127.64.0/18

```

This rule does the same thing but for packets coming from the other cluster. It maps the packets src IP using the remapped cidr. this is necessary to route the returning packets. 

```yaml
- match:
    - dev:
        position: out
        value: eth0
      op: neq
    - ip:
        position: src
        value: 10.127.64.0/18
      op: eq
    - dev:
        position: in
        value: liqo-tunnel
      op: eq
  name: 75b6467c-4ce3-4434-8e18-9b4f568c12c7
  natType: snat
  to: 10.71.0.0/18

```

### \<tenant-name\>-remap-externalcidr (Gateway)

It works like **\<tenant-name\>-remap-podcidr** for **external-cidr**.

### \<name\>-remap-ipmapping-gw

These fwcfgs are created from IP resources (check the IP paragraph), containing SNAT rules and DNAT rules to make the "local IP" reachable through the external-cidr. 

# Route Configuration

Route configurations use policy routing only. 

## Labels

TODO

## Before Peering

### \<local-cluster-id\>-\<node-name\>-extcidr (Gateway)

It contains all the routes that match the traffic targeting the local extcidr.

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: RouteConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/route-category: gateway
    networking.liqo.io/route-subcategory: fabric-node
    networking.liqo.io/route-unique: cheina-cluster1-worker
  name: cheina-cluster1-worker-extcidr
  namespace: liqo
spec:
  table:
    name: cheina-cluster1-worker-extcidr
    rules:
    - iif: liqo-tunnel
      routes:
      - dev: liqo.cjntnn4bdj
        dst: 10.111.105.133/32
        gw: 10.80.0.3
      - dev: liqo.cjntnn4bdj
        dst: 10.111.0.1/32
        gw: 10.80.0.3
```

### \<local-cluster-id\>-\<node-name\>-service-nodeport-routing (Gateway)

These rules use marks with policy routing to route the returning traffic towards the correct node. Check **service-nodeport-routing** firewall configuration for more insight.

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: RouteConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/route-category: gateway
    networking.liqo.io/route-subcategory: fabric
  name: cheina-cluster1-control-plane-service-nodeport-routing
  namespace: liqo
spec:
  table:
    name: cheina-cluster1-control-plane-service-nodeport-routing
    rules:
    - dst: 10.70.0.0/32
      fwmark: 2
      routes:
      - dev: liqo.jdr5xndgmb
        dst: 10.70.0.0/32
        gw: 10.80.0.2
      targetRef:
        kind: InternalNode
        name: cheina-cluster1-control-plane
```

### \<local-cluster-id\>-\<node-name\>-gw-node (Gateway)

It contains the rule that allow traffic from gateway to nodes using geneve tunnels. Please notice that Liqo uses the internal CIDR to assign an IP to every geneve interfaces. If you need to debug the traffic between geneve interfaces you can ping each interface, This is the first rule in the

It also contains a route for each pod in the cluster. These routes allows the traffic coming from other clusters to be forwarded in the correct node. This is necessary because kubernetes does not provide a standard way to understand the podcidr range used for each node.

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: RouteConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/route-category: gateway
    networking.liqo.io/route-subcategory: fabric
  name: cheina-cluster1-control-plane-gw-node
  namespace: liqo
spec:
  table:
    name: cheina-cluster1-control-plane
    rules:
      - dst: 10.80.0.2/32
        routes:
          - dev: liqo.jdr5xndgmb
            dst: 10.80.0.2/32
            scope: link
      - iif: liqo-tunnel
        routes:
          - dst: 10.112.0.229/32
            gw: 10.80.0.2
            targetRef:
              kind: Pod
              name: coredns-9ff4c5cf6-xbx5w
              namespace: kube-system
              uid: 3cb83b91-98b5-412c-b5a2-f1ebe28497df
```

### \<local-cluster-id\>-gw-ext (Gateway)

This rtcfg contains all the routes that forward the traffic from a gateway to another. It contains the rules for remote podcidrs and extcidrs.

Please notice that the routes with destination 10.70.0.0/16 is related to excidr. It should  seem strange since it is not using a remapped cidr. This is because the DNAT rules (which translate from remapped cidr to original cidr) acts in **prerouting**.

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: RouteConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/route-category: gateway
    networking.liqo.io/route-unique: cheina-cluster2
  name: cheina-cluster2-gw-ext
  namespace: liqo-tenant-cheina-cluster2
spec:
  table:
    name: cheina-cluster2
    rules:
    - dst: 10.122.0.0/16
      iif: liqo.fgckffk4dv
      routes:
      - dst: 10.122.0.0/16
        gw: 169.254.18.1
    - dst: 10.70.0.0/16
      iif: liqo.fgckffk4dv
      routes:
      - dst: 10.70.0.0/16
        gw: 169.254.18.1
    - dst: 10.122.0.0/16
      iif: liqo.jdr5xndgmb
      routes:
      - dst: 10.122.0.0/16
        gw: 169.254.18.1
    - dst: 10.70.0.0/16
      iif: liqo.jdr5xndgmb
      routes:
      - dst: 10.70.0.0/16
        gw: 169.254.18.1
    - dst: 10.122.0.0/16
      iif: liqo.cjntnn4bdj
      routes:
      - dst: 10.122.0.0/16
        gw: 169.254.18.1
    - dst: 10.70.0.0/16
      iif: liqo.cjntnn4bdj
      routes:
      - dst: 10.70.0.0/16
        gw: 169.254.18.1
```

### \<local-cluster-id\>-node-gw (Node)

This rtcfg  contains the routes to reach the "other" side of the geneve tunnel. It also includes all the routes that point to the remote cluster's podcidr and extcidr. 

```yaml
apiVersion: networking.liqo.io/v1beta1
kind: RouteConfiguration
metadata:
  labels:
    liqo.io/managed: "true"
    networking.liqo.io/route-category: fabric
  name: cheina-cluster2-node-gw
  namespace: liqo-tenant-cheina-cluster2
spec:
  table:
    name: cheina-cluster2-node-gw
    rules:
    - dst: 10.80.0.4/32
      routes:
      - dev: liqo.7hr82v9br5
        dst: 10.80.0.4/32
        scope: link
    - dst: 10.68.0.0/16
      routes:
      - dst: 10.68.0.0/16
        gw: 10.80.0.4
    - dst: 10.71.0.0/18
      routes:
      - dst: 10.71.0.0/18
        gw: 10.80.0.4
```

# Debug commands

- tcpdump -tnl -i any <protocol>
- tcpdump -tnl -i any tcp port 8080
- tcpdump -tnl -i any tcp dst port 8080
