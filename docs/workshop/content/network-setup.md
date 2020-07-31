In this section we're going to be configuring the networking for our environment. 

With OpenShift virtualisation we have a few different options for networking - we can just have our virtual machines be attached to the same pod networks that our containers would have access to, or we can configure more real-world virtualisation networking constructs like bridged networking, SR/IOV, and so on. It's also absolutely possible to have a combination of these, e.g. both pod networking and a bridged interface directly attached to a VM at the same time, using Multus, the default networking CNI in OpenShift 4.x.

In this lab we're going to enable multiple options - pod networking and a secondary network interface provided by a bridge on the underlying worker nodes (hypervisors). Each of the worker nodes has been configured with an additional, currently unused, network interface that is defined as `ens4`, and we'll need a bridge device, `br1` to be created so we can attach our virtual machines to it. The first step is to use the new Kubernetes NetworkManager state configuration to setup the underlying hosts to our liking. Recall that we can get the **current** state by requesting the `NetworkNodeState`:

~~~bash
$ oc get nodes

NAME                                STATUS   ROLES    AGE    VERSION
cluster-august-lhrd5-master-0       Ready    master   177m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-1       Ready    master   177m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-2       Ready    master   177m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-6w624   Ready    worker   158m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-mh52l   Ready    worker   159m   v1.18.3+b74c5ed
~~~

~~~bash
$ oc get nns/cluster-august-lhrd5-worker-mh52l -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2020-07-24T02:47:44Z"
  generation: 1
  name: cluster-august-lhrd5-worker-mh52l
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: cluster-august-lhrd5-worker-mh52l
    uid: a9bd32e8-2938-4ddb-95ca-4d022b3e044f
  resourceVersion: "94081"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkstates/cluster-august-lhrd5-worker-mh52l
  uid: fa3d66bc-aa88-4d37-8e4b-49cc27b66e02
status:
  currentState:
    dns-resolver:
      config:
        search: []
        server: []
      running:
        search:
        - cluster-august.students.osp.opentlc.com
        - cluster-august.students.osp.opentlc.com
        server:
        - 10.0.0.10
        - 10.0.0.11
        - 10.0.0.12
        - 192.168.47.11
        - 192.168.47.12
        - 192.168.47.10
    interfaces:
    - ipv4:
        enabled: false
      ipv6:
        enabled: false
      mtu: 1450
      name: br0
      state: down
      type: ovs-interface
    - ipv4:
(...)
~~~

In there you'll spot the interface that we'd like to use to create a bridge, `ens7`, ignore the IP address that it has right now, that just came from DHCP in our environment.

~~~bash
    - ipv4:
        address:
        - ip: 192.168.47.14
          prefix-length: 24
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        address:
        - ip: fe80::6659:ee6e:7ecd:ca28
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      mac-address: FA:16:3E:FA:C0:61
      mtu: 1500
      name: ens7
      state: up
      type: ethernet
~~~

> **NOTE**: The first interface `ens3` via `br0` is being used for inter-OpenShift communication, including all of the pod networking via OpenShift SDN.

Now, one thing to also check if all your nodes have the *same* interface naming. Let's check ours:

~~~bash
$ oc get nns/cluster-august-lhrd5-worker-6w624 -o yaml | grep ens | grep name
      name: ens3
      name: ens6
$ oc get nns/cluster-august-lhrd5-worker-mh52l -o yaml | grep ens | grep name
      name: ens3 <--- On the other node the first interface is also ens3 which is good.
      name: ens7 <--- On the other node the second interface is ens6! This means we have a difference.
~~~

Uh Oh! This shouldn't happen, but if it does you'll need to adjust some of your next steps to apply different policies to each node. 

Now we can apply a new `NodeNetworkConfigurationPolicy` for our worker nodes to setup a desired state for `br1` via the secondary nic. 

### Same NIC naming across nodes

If your workers have all the same nic config naming then you can specify a nodeSelector like this: `node-role.kubernetes.io/worker: ""` This applies the **same** config to all worker nodes.

REMINDER: If you found you have different naming, as we do in the example above, you'll need to apply different policies, one for each node. If this is the case then skip to [this section now](#different-nic-names) now. Otherwise continue with the example below:

**Again, this example covers all workers with the same nic.** In this example it is `ens4` but adjust it to fit yours:

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

Now proceed to [the next step](#review-the-policy) to review the policy you just set.

###Different NIC names

If, like in our example here, we have two different nic names (ens6 on cluster-august-lhrd5-worker-6w624 and ens7 on cluster-august-lhrd5-worker-mh52l) you'll need two policies, one for each configuration. First, we need to label the nodes, so we an easily reference them:

> **NOTE**: Reminder, if your deployment has the same NIC naming, you don't need to do the next few steps. If this is the case skip to [the next step](#review-the-policy).

~~~bash
$ oc label node cluster-august-lhrd5-worker-6w624 nic2=ens6
node/cluster-august-lhrd5-worker-6w624 labeled

$ oc label node cluster-august-lhrd5-worker-mh52l nic2=ens7
node/cluster-august-lhrd5-worker-mh52l labeled
~~~

This sets a key of `nic2` for each node and supplies the right value to that `key`.

> **NOTE**: There are actually a few ways you could do this. For instance, instead of a label, we could use the built-in node labels `kubernetes.io/hostname` and simply assign `kubernetes.io/hostname=cluster-august-lhrd5-worker-6w624`. 

Now, we can create unique NNCP's for each node.

Be sure to adjust the port name for the correct nic name as well as the description for clarity. In the below examples we are asking for the policy to match ANY workers (`node-role.kubernetes.io/worker: ""`) with the specific label. This ensures other node types aren't affected.

ens6 on cluster-august-lhrd5-worker-6w624

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-ens6-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
    nic2: ens6
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with ens6 as a port
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
            - name: ens6
EOF
~~~

ens7 on cluster-august-lhrd5-worker-mh52l

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-ens7-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
    nic2: ens7
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with ens7 as a port
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
            - name: ens7
EOF
~~~

###Review the policy

Then enquire as to whether it was successfully applied:

~~~bash
$ oc get nncp
NAME                      STATUS
br1-ens6-policy-workers   SuccessfullyConfigured
br1-ens7-policy-workers   SuccessfullyConfigured

$ oc get nnce
NAME                                                        STATUS
cluster-august-lhrd5-worker-6w624.br1-ens6-policy-workers   SuccessfullyConfigured
cluster-august-lhrd5-worker-mh52l.br1-ens7-policy-workers   SuccessfullyConfigured
~~~

We can also dive into the `NetworkNodeConfigurationPolicy` (**nncp**) a little further:

~~~bash
$ oc get nncp/br1-ens7-policy-workers -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"nmstate.io/v1alpha1","kind":"NodeNetworkConfigurationPolicy","metadata":{"annotations":{},"name":"br1-ens7-policy-workers"},"spec":{"desiredState":{"interfaces":[{"bridge":{"options":{"stp":{"enabled":false}},"port":[{"name":"ens7"}]},"description":"Linux bridge with ens7 as a port","ipv4":{"dhcp":true,"enabled":true},"name":"br1","state":"up","type":"linux-bridge"}]},"nodeSelector":{"nic2":"ens7","node-role.kubernetes.io/worker":""}}}
  creationTimestamp: "2020-07-24T04:53:52Z"
  generation: 1
  name: br1-ens7-policy-workers
  resourceVersion: "124545"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkconfigurationpolicies/br1-ens7-policy-workers
  uid: db861cce-3674-4ca0-bceb-d83a3fe9b231
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens7
      description: Linux bridge with ens7 as a port
      ipv4:
        dhcp: true
        enabled: true
      name: br1
      state: up
      type: linux-bridge
  nodeSelector:
    nic2: ens7
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - lastHearbeatTime: "2020-07-24T04:54:04Z"
    lastTransitionTime: "2020-07-24T04:54:04Z"
    reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
  - lastHearbeatTime: "2020-07-24T04:54:04Z"
    lastTransitionTime: "2020-07-24T04:54:04Z"
    message: 1/1 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
~~~

> **NOTE**: You might also like to try going onto each node with `oc debug node/node_name` and running an `ip a`. You'll find br1 holding the node's IP! 

Now that the "physical" networking is configured on the underlying worker nodes, we need to then define a `NetworkAttachmentDefinition` so that when we want to use this bridge, OpenShift and OpenShift virtualisation know how to attach into it. This associates the bridge we just defined with a logical name, known here as '**tuning-bridge-fixed**':

> **NOTE**: Since we named the bridge `br1` on both nodes in both our NodeNetworkConfigurationPolicy's we don't need to create multiple versions of this `NetworkAttachmentDefinition`.

~~~bash
$ cat << EOF | oc apply -f -
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