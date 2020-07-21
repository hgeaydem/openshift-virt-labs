In this section we're going to be configuring the networking for our environment. 

With OpenShift virtualisation we have a few different options for networking - we can just have our virtual machines be attached to the same pod networks that our containers would have access to, or we can configure more real-world virtualisation networking constructs like bridged networking, SR/IOV, and so on. It's also absolutely possible to have a combination of these, e.g. both pod networking and a bridged interface directly attached to a VM at the same time, using Multus, the default networking CNI in OpenShift 4.x.

In this lab we're going to enable multiple options - pod networking and a secondary network interface provided by a bridge on the underlying worker nodes (hypervisors). Each of the worker nodes has been configured with an additional, currently unused, network interface that is defined as `ens4`, and we'll need a bridge device, `br1` to be created so we can attach our virtual machines to it. The first step is to use the new Kubernetes NetworkManager state configuration to setup the underlying hosts to our liking. Recall that we can get the **current** state by requesting the `NetworkNodeState`:

~~~bash
$ oc get nodes
NAME                     STATUS   ROLES    AGE   VERSION
ocp-9pv98-master-0       Ready    master   20h   v1.18.3+b74c5ed
ocp-9pv98-master-1       Ready    master   20h   v1.18.3+b74c5ed
ocp-9pv98-master-2       Ready    master   20h   v1.18.3+b74c5ed
ocp-9pv98-worker-g78bj   Ready    worker   20h   v1.18.3+b74c5ed
ocp-9pv98-worker-pj2dn   Ready    worker   20h   v1.18.3+b74c5ed
~~~

~~~bash
$ oc get nns/ocp-9pv98-worker-g78bj -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2020-07-21T00:52:52Z"
  generation: 1
  name: ocp-9pv98-worker-g78bj
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: ocp-9pv98-worker-g78bj
    uid: e2b20d97-4c7b-49f4-9596-3acfb8c8c835
  resourceVersion: "449091"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkstates/ocp-9pv98-worker-g78bj
  uid: cc8b1eb9-62d0-44d1-acdd-b6130a8d6138
status:
  currentState:
    dns-resolver:
      config:
        search: []
        server: []
      running:
        search:
        - ocp.augusts.be
        - ocp.augusts.be
        server:
        - 10.0.0.10
        - 10.0.0.11
        - 10.0.0.12
        - 192.168.0.21
        - 192.168.0.22
        - 192.168.0.20
    interfaces:
    - ipv4:

...
~~~

In there you'll spot the interface that we'd like to use to create a bridge, `ens4`, ignore the IP address that it has right now, that just came from DHCP in our environment.

~~~bash
    - ipv4:
        address:
        - ip: 192.168.0.29
          prefix-length: 24
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        address:
        - ip: fe80::51e9:83ad:40d6:643c
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      mac-address: FA:16:3E:69:DE:31
      mtu: 1500
      name: ens4
      state: up
      type: ethernet
~~~

> **NOTE**: The first interface `ens3` via `br0` is being used for inter-OpenShift communication, including all of the pod networking via OpenShift SDN.

Now we can apply a new `NodeNetworkConfigurationPolicy` for our worker nodes to setup a desired state for `br1` via `ens4`, noting that in the `spec` we specify a `nodeSelector` to ensure that this **only** gets applied to our worker nodes:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-ens4-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with ens4 as a port
        type: linux-bridge
        state: up
        ipv4:
          dhcp: true
          enabled: true
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: ens4
EOF

nodenetworkconfigurationpolicy.nmstate.io/br1-ens4-policy-workers created
~~~

Then enquire as to whether it was successfully applied:

~~~bash
$  oc get nncp
NAME                      STATUS
br1-ens4-policy-workers   SuccessfullyConfigured

$ oc get nnce
NAME                                             STATUS
ocp-9pv98-master-0.br1-ens4-policy-workers       NodeSelectorNotMatching
ocp-9pv98-master-1.br1-ens4-policy-workers       NodeSelectorNotMatching
ocp-9pv98-master-2.br1-ens4-policy-workers       NodeSelectorNotMatching
ocp-9pv98-worker-g78bj.br1-ens4-policy-workers   SuccessfullyConfigured
ocp-9pv98-worker-pj2dn.br1-ens4-policy-workers   SuccessfullyConfigured
~~~

We can also dive into the `NetworkNodeConfigurationPolicy` (**nncp**) a little further:

~~~bash
$ oc get nncp/br1-ens4-policy-workers -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"nmstate.io/v1alpha1","kind":"NodeNetworkConfigurationPolicy","metadata":{"annotations":{},"name":"br1-ens4-policy-workers"},"spec":{"desiredState":{"interfaces":[{"bridge":{"options":{"stp":{"enabled":false}},"port":[{"name":"ens4"}]},"description":"Linux bridge with ens4 as a port","ipv4":{"dhcp":true,"enabled":false},"name":"br1","state":"up","type":"linux-bridge"}]},"nodeSelector":{"node-role.kubernetes.io/worker":""}}}
  creationTimestamp: "2020-07-21T02:46:49Z"
  generation: 1
  name: br1-ens4-policy-workers
  resourceVersion: "451443"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkconfigurationpolicies/br1-ens4-policy-workers
  uid: c0180f2a-f001-4d50-9145-11f7436c78bf
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens4
      description: Linux bridge with ens4 as a port
      ipv4:
        dhcp: true
        enabled: false
      name: br1
      state: up
      type: linux-bridge
  nodeSelector:
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - lastHearbeatTime: "2020-07-21T02:47:00Z"
    lastTransitionTime: "2020-07-21T02:47:00Z"
    reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
  - lastHearbeatTime: "2020-07-21T02:47:00Z"
    lastTransitionTime: "2020-07-21T02:47:00Z"
    message: 2/2 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
~~~


Now that the "physical" networking is configured on the underlying worker nodes, we need to then define a `NetworkAttachmentDefinition` so that when we want to use this bridge, OpenShift and OpenShift virtualisation know how to attach into it. This associates the bridge we just defined with a logical name, known here as '**tuning-bridge-fixed**':

~~~bash
cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: tuning-bridge-fixed
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "groot",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br1"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF

networkattachmentdefinition.k8s.cni.cncf.io/tuning-bridge-fixed created
~~~

> **NOTE**: The important flags to recognise here are the **type**, being **cnv-bridge** which is a specific implementation that links in-VM interfaces to a counterpart on the underlying host for full-passthrough of networking. Also note that there is no **ipam** listed - we don't want the CNI to manage the network address allocation for us - the network we want to attach to has DHCP enabled, and so let's not get involved.