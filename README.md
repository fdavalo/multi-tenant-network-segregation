# micro segmentation on Openshift

## Utilisation des CNI Openshift (Openshift SDN, OVN Kubernetes)

### Gestion des _Network Policy_

* Console Openshift : 

    Depuis la 4.10, il y a déjà dans la console un onglet dédié aux network policies avec un formulaire pour les créer (ajout de pod selector, d'ingress ou d'egress rule) et la possibilité de voir une prévisualisation des pods affectés par cette policy.

* Opérateur Network Observability :

    Il y a aussi depuis la 4.10 en DevPreview, l'opérateur Network Observability qui permet (sur OVN) de collecter et visualiser (console + dashboard) les flux réseaux du cluster.  (The eBPF agent is not officially released yet, it is already provided as a preview.)
    
    En 4.11, cet opérateur passera en techPreview et utilisera eBPF pour collecter les flux et rendra l'opérateur indépendant du CNI installé.
    
    [![Observability](https://github.com/fdavalo/microseg/blob/main/network-observability.png?raw=true)](network-observability.png)

* Advanced Cluster Security (RHACS) :
 
   Sur la version actuelle de RHACS, il y a déjà une gestion des network policy : 
   
     * capture et visualisation des flux (graphiquement + types de flux acceptés, ingress, interdits, filtres, etc ...)
             
        [![Graphes](https://github.com/fdavalo/microseg/blob/main/acs-netpol-graph.png?raw=true)](netpol-graph.png)
     
     * import de network policy pour simulation 

     	[![Simulation](https://github.com/fdavalo/microseg/blob/main/acs-netpol-sim.png?raw=true)](acs-netpol-sim.png)

     * génération de network policy
     
     * ajout possible d'une politique de vérification/application d'une network policy a minima dans un namespace

    `Kubernetes network policies are vital in helping to enable zero-trust networking within a cluster. They reduce the impact of network attacks by limiting the opportunity for lateral movement, whereas Kubernetes resources are not isolated by default. Many, if not most of our customers, find this technology to be challenging and often end up with misconfigured or missing network policies. RHACS 3.70 helps security teams take the first step by identifying missing network policies altogether.`
    
    More specifically, RHACS  can now identify all deployments that are not selected by any ingres  network policy in the namespace, and includes a system policy to trigger against such deployments.  


* Cilium editor

    Il y a aussi un éditeur de network policy proposé par Cilium, qui nécessite l'import des flux réels collectés par hubble sur votre cluster, on retrouve les fonctionnalités de RHACS + Openshift.
    
    Je n'ai pas eu le temps de tester la collecte des flux via Hubble sur Openshift ou de reformater l'export de l'opérateur Network Observability au format hubble.

* Gestion logique de la micro segmentation des applications

    Sinon, il y a un important client Openshift mais qui a créé son propre référentiel de flux logiques et génère les network policy associées. (et autres éléments potentiels, ingress IP, egress IP, ....)

    Je regarde s'il existe un outil open source pour déclarer namespace et les flux associés.

### Ingress sharding / Ingress IP

   Sinon, il y avait une question sur l'utilisation d'un routeur (shard) par namespace , utilisant une ingress IP (via un service géré par MetalLB) pour chaque routeur.
 
   Ce modèle de segmentation fonctionne bien en bare metal, metal LB permettra une bascule des IP en mode L2 ou BGP pour garder la résilience.
    
   _Après avoir consulté l'ingénierie, ils n'ont pas à leur connaissance de limitation sur le nombre de routeurs shardés, la seule limitation reste le nombre de routes par routeur (max 1000)._
   
[![Ingress IP](https://github.com/fdavalo/microseg/blob/main/ingress-ip.png?raw=true)](ingress-ip.png)

### Egress IP

   Au niveau des egress IP par namespace, il est possible de définir une ou plusieurs egress IP par namespace, pour permettre d'identifier les flux sortant d'un namespace à partir de leur egress IP et pour permettre de rattacher des politiques sur les firewall.

   As a cluster administrator, you can configure the OVN-Kubernetes Container Network Interface (CNI) cluster network provider to assign one or more egress IP addresses to a namespace
   
[![Egress IPs](https://github.com/fdavalo/microseg/blob/main/egress-ips.png?raw=true)](egress-ips.png)

### slides en référence

   Pour finir, je vous joins un certain nombre de slides qui donnent plus de précisions sur les fonctionnalités des CNI dans Openshift.

Openshift Networking: https://github.com/fdavalo/microseg/blob/main/Openshift%20Networking.pdf?raw=true

Network security for apps on Openshift : https://www.redhat.com/files/summit/session-assets/2018/Network-security-for-apps-on-OpenShift.pdf

An Implementation of Red Hat OpenShift Network Isolation Using Multiple Ingress Controllers : https://www.redbooks.ibm.com/redpapers/pdfs/redp5641.pdf
