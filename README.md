# skupper hazelcast demo

Simple example using Skupper v2 to extend a hazelcast cluster across sites.

Relies on hazelcast members using the Kube API to discover:
* peer members in the same namespace through a "hazelcast" service.
* peer members in remote sites exposed via a skupper listener with exposePodsByName set

# Setup

I have named two sites "boston" and "madrid". This was important (for now) so
that we could guess the names for labeling the skupper services.

**Using the "boston" Cluster**
```
kubectl apply -f boston.yaml
```

**Using the "madrid" Cluster**
```
kubectl apply -f madrid.yaml
```

Adds the following resources to the namespace:
* Skupper Site
* Skupper Connector exposing the local member workload to the skupper network
  with a unique routing key per site with **exposePodsByName** enabled.
* Skupper Listener for the remote site's hazelcast members with
  **exposePodsByName** enabled.
* A Hazelcast StatefulSet and configuration. The need for a statefulset as a workaround for...
* A collection of ConfigMaps with the `skupper.io/label-template` label to
  instruct the skupper controller to apply the discovery label to specific
  services by name exact match.


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

Once everything stabilizes you should see cluster member discovery complete. It
seems like discovery "gives up" after some time, so you may need to restart the
members on one of the sites once they are linked.
