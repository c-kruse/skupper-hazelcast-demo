# skupper hazelcast demo

Simple example using Skupper v2 to connect a hazelcast client running on one
site to a hazelcast cluster running in a remote site.

# Setup

**Using the Client Kubernetes Cluster**
```
kubectl apply -f client.yaml
```

Adds the following resources to the namesapce:
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

Adds the following resources to the namesapce:
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

# In the skupper network console
![skupper network topology with hazelcast](./resources/SkupperConsoleHazelcast.png)
