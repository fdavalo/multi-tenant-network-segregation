# multi-tenant network segregation on Openshift

## What we want to implement

Schemas

[![Schema 1](https://github.com/fdavalo/multi-tenant-network-segregation/blob/main/tenant-seg-flow3.png?raw=true)](tenant-seg-flow3.png)

[![Schema 2](https://github.com/fdavalo/multi-tenant-network-segregation/blob/main/tenant-seg-flow1.png?raw=true)](tenant-seg-flow1.png)

## Example of tenant segregation by subnet and by namespaces

The goal here is to segregate at maximum between tenant

 * Specific subnet for each tenant
   * 10.6.85.0/24 = tenant1085
   * 10.6.81.0/24 = tenant1081
   
 * Dedicated namespaces to specific tenants
   * tenan1085-ns1 and tenant1085-ns2 will host workload for tenant 1085
   * tenant1081-ns1 host workload for tenant 1081
 
 [![namespaces](https://github.com/fdavalo/multi-tenant-network-segregation/blob/main/tenant-ns.png?raw=true)](tenant-ns.png)
 
 * We use a **NodeNetworkConfigurationPolicy** object to add a tagged vlan from the trunked secondary interfaces in the cluster nodes
   we can also add specific tenant IPs for some nodes and use those IPs as external ingress IPs (see next chapter)
   
   * nodes with a specific IP by tenant provisioned 
   
            apiVersion: nmstate.io/v1
            kind: NodeNetworkConfigurationPolicy
            metadata:
              name: vlan-ens224-worker-zqffn-policy
            spec:
              desiredState:
                interfaces:
                  - description: VLAN 1085 using ens224
                    ipv4:
                      address:
                        - ip: 10.6.85.197
                          prefix-length: 24
                      dhcp: false
                      enabled: true
                    name: ens224.1085
                    state: up
                    type: vlan
                    vlan:
                      base-iface: ens224
                      id: 1085
              nodeSelector:
                kubernetes.io/hostname: ocp1-bm4nq-worker-zqffn   
   
   * nodes without a specific IP by tenant

            apiVersion: nmstate.io/v1
            kind: NodeNetworkConfigurationPolicy
            metadata:
              name: vlan-ens224-worker-vqsrj-policy
            spec:
              desiredState:
                interfaces:
                  - description: VLAN 1085 using ens224
                    ipv4:
                      dhcp: false
                      enabled: false
                    name: ens224.1085
                    state: up
                    type: vlan
                    vlan:
                      base-iface: ens224
                      id: 1085
              nodeSelector:
                kubernetes.io/hostname: ocp1-bm4nq-worker-vqsrj
    
How we segregate ingress flows

 * A specific ingresscontroller by tenant

              apiVersion: operator.openshift.io/v1
              kind: IngressController
              metadata:
                name: tenant1085
                namespace: openshift-ingress-operator
              spec:
                ...
                domain: tenant1085.ocp1.redhat.hpecic.net
                ...
                namespaceSelector:
                  matchLabels:
                    tenant: tenant1085
                replicas: 1
                ...
                endpointPublishingStrategy:
                  loadBalancer:
                    scope: External
                  type: LoadBalancerService
                  
 * Each tenant ingress servers only routes from tenant namespaces (using label **tenant** in namespaces)

            kind: Namespace
            apiVersion: v1
            metadata:
              name: tenant1085-ns1
              labels:
                tenant: tenant1085

 * Modify the default ingress controller, not to watch routes from tenant namespaces  
    
                 namespaceSelector:
                   matchExpressions:
                     - key: tenant
                       operator: NotIn
                       values:
                         - tenant1085
                         - tenant1081    
    
 * A dedicated VIP for each tenant subnet to access the tenant ingress pods
   * VIP 10.6.85.10:80 for tenant1085 (haproxy) loadbalances to 10.6.85.196:80 and 10.6.85.197:80
     10.6.85.196 and 10.6.85.197 are external IPs for the LoadBalancer service that targets the tenant ingress pods  
   * VIP 10.6.81.10:80 for tenant1081 (haproxy) loadbalances to 10.6.81.196:80 and 10.6.81.197:80
     10.6.81.196 and 10.6.81.197 are external IPs for the LoadBalancer service that targets the tenant ingress pods

How we segregate egress flows

 * A **NetworkAttachementDefinition** is created to enable inserting a tenant interface on pods
   * we use here ipvlan (macvlan needs vsphere portgroup configuration) and dhcpd is handled in openshift with whereabouts service
   * we could use macvlan instead and use DHCP from the network

              apiVersion: k8s.cni.cncf.io/v1
              kind: NetworkAttachmentDefinition
              metadata:
                name: ipvlan-1085
                namespace: tenant1085-ns1
              spec:
                config: |-
                  {
                    "cniVersion": "0.3.1",
                    "name": "ipvlan-1085",
                    "type": "ipvlan",
                    "master": "ens224.1085",
                    "mode": "l2",
                    "ipam": {
                        "type": "whereabouts",
                        "range": "10.6.85.0/24",
                        "range_start": "10.6.85.221",
                        "range_end": "10.6.85.230"
                    }
                  }

 * Each deployment that need egress flows have to add a multus annotation to use the tenant subnet and the tenant default gateway

              kind: Deployment
              apiVersion: apps/v1
              metadata:
                ...
              spec:
                ...
                template:
                  metadata:
                    annotations:
                      k8s.v1.cni.cncf.io/networks: '[{ "name": "ipvlan-1085", "default-route": ["10.6.85.1"]} ]'


 * External flows will go thru the tenant default gateway, the flows will originate from a tenant subnet IP (ex: 10.6.85.221, pod IP)
   * those flows will then be secured using traditional firewalls, if needed

How we segregate internal flows

 * **NetworkPolicies** can secure inter-namespaces flows

Implementation

 * See manifests directory with kubernetes objects configuration
 * See notes.md for more specific implementations

Documentation

 * See docs.md for more information on Network and Openshift
 
 * See gatekeeper.md for the details on how to set Network Policies and use GateKeeper to enforce it
 
  [![namespaces](https://github.com/fdavalo/multi-tenant-network-segregation/blob/main/gatekeeper.md?raw=false)](gatekeeper.md)
