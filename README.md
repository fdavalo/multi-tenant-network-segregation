# multi-tenant network segregation on Openshift

## Example of tenant segregation by subnet and by namespaces

The goal here is to segregate at maximum between tenant

 * Specific subnet for each tenant
   * 10.6.85.0/24 = tenant1085
   * 10.6.81.0/24 = tenant1081
 * We use a **NodeNetworkConfigurationPolicy** object to add a tagged vlan from the trunked secondary interfaces in the cluster nodes
   we can also add specific tenant IPs for some nodes and use those IPs as external ingress IPs (see next chapter)

How we segregate ingress flows

 * Specific ingresscontroller by tenant
 * A dedicated VIP for each tenant subnet to access the tenant ingress pods
   * VIP 10.6.85.10:80 for tenant1085 (haproxy) loadbalances to 10.6.85.196:80 and 10.6.85.197:80
     10.6.85.196 and 10.6.85.197 are external IPs for the LoadBalancer service that targets the tenant ingress pods  
   * VIP 10.6.81.10:80 for tenant1081 (haproxy) loadbalances to 10.6.81.196:80 and 10.6.81.197:80
     10.6.81.196 and 10.6.81.197 are external IPs for the LoadBalancer service that targets the tenant ingress pods
 * Each tenant ingress servers only routes from tenant namespaces (using label **tenant** in namespaces)

How we segregate egress flows

 * Each deployment that need egress flows have to add a multus annotation to use the tenant subnet and the tenant default gateway
 * A **NetworkAttachementDefinition** is created to enable inserting a tenant interface on pods
   * we use here ipvlan (macvlan needs vsphere portgroup configuration) and dhcpd is handled in openshift with whereabouts service
   * we could use macvlan instead and use DHCP from the network
 * External flows will go thru the tenant default gateway, the flows will originate from a tenant subnet IP (ex: 10.6.85.221, pod IP)
   * those flows will then secured using traditional firewalls, if needed

How we segregate internal flows

 * **NetworkPolicies** can secure inter-namespaces flows

Implementation

 * See manifests directory with kubernetes objects configuration
 * See notes.md for more specific implementations
