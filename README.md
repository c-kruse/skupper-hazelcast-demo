# skupper hazelcast demo

Simple example using Skupper v2 to connect a hazelcast client running on one
site to a hazelcast cluster running in a remote site.

# Setup

**Using the Client Kubernetes Cluster**
```
kubectl apply -f client.yaml
```

Adds the following resources to the namespace:
* Skupper Site
* Skupper Listener for the hazelcast service with **exposePodsByName** enabled.
* Hazelcast management center deployment with with a client configuration using
  standard kubernetes dns-based service discovery
* A LoadBalancer service to expose it outside of the cluster (optional)

**Using the Hazelcast Cluster Kubernetes Cluster**
```
NAMESPACE=$(kubectl config view --output 'jsonpath={..namespace}')
sed "s/MY_NAMESPACE/$NAMESPACE/" cluster.yaml | kubectl apply -f -
```

Adds the following resources to the namespace:
* Skupper Site
* Skupper Connector for the hazelcast statefulset with **exposePodsByName** enabled.
* Hazelcast cluster StatefulSet with a special configuration to enable
  discovery and addressing that works across networks.
  * Sets `hazelcast.network.public-address` explicitly to the name of the pod/hostname.
  * Uses standard kuberentes service based DNS discovery.
  * Adjusted pod dnsConfig to resolve peer members though k8s pod networking
    using the statefulSet Serivce **hack**. Allows pods to resolve
    "hazelcast-0" to "hazelcast-0.hazelcast.<my-namespace>.svc.cluster.local"
* Headless service for statefulset

# Linking the sites

Using the skupper 2.0 CLI,

In one of the sites that is accessible externally via LB or route.
```
skupper token issue accesstoken.yaml
```

In the other site
```
skupper token redeem accesstoken.yaml
```

# See Working Cluster

Once everything stabilizes you can open the management center  from the client site to see everything working.
```
kubectl describe service hazelcast-mancenter
```

*note it seems like management center has difficulty recovering after discovery
initially fails before the sites are linked. It may need restarted (kubectl
rollout restart deployment/hazelcast-mancenter).

## View from hazelcast management center
![hazelcast cluster members from external site](./resources/HazelcastManagementCenterView.png)
## View from the skupper network console
![skupper network topology with hazelcast](./resources/SkupperConsoleHazelcast.png)

# Limitations

* Requires a StatefulSet.

Hazelcast discovery works by sharing cluster member's "public-address". By
default, this will be the pod bind address - an IP in the kube pod network.
When using skupper to access the hazelcast cluster members without sharing a
pod network, we need a to use a different addressing scheme. This method uses
the cluster member's pod name. On the Listener-side over the **skupper**
network this just magically works out with the exposePodsByName option, but to
make this work over the **pod network** we rely on a trick - using a
StatefulSet and service so that k8s will provision DNS records for each pod
under <podname>.<servicename>.<namespace>.svc.cluster.local.

An alternative is to simply disable "smart routing" on the client side, and to
not set exposePodsByName. In this configuration, the Hazelcast cluster members
can use their default behaviour to resolve each other over the kube pod
network, and the client will allow skupper to simply loadbalance connections to
any cluster members (called unisocket client connections in hazelcast.)

* Only accommodates a client/server model.

Does not work for extending a hazelcast cluster across sites. It is unclear if
this is desirable, hazelcast documentation seems to suggest it may not be idea.
