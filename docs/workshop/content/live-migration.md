Live Migration is the process of moving an instance from one node in a cluster to another without interruption. This process can be manual or automatic. This depends if the `evictionStrategy` strategy is set to `LiveMigrate` and the underlying node is placed into maintenance. It also depends on the storage access mode. Virtual machines must have a PersistentVolumeClaim (PVC) with a shared ReadWriteMany (RWX) access mode to be live migrated.

Live migration is an administrative function in OpenShift Virtualization. While the action is visible to all users, only admins can initiate a migration. Migration limits and timeouts are managed via the `kubevirt-config` `configmap`. For more details about limits see the [documentation](https://docs.openshift.com/container-platform/4.4/cnv/cnv_live_migration/cnv-live-migration-limits.html#cnv-live-migration-limits).

In our lab you should now have only one VM running. You can check that, and view the underlying host it is on, by looking at the virtual machine's instance with the `oc get vmi` command.

> **NOTE**: In OpenShift Virtualization, the "Virtual Machine" object can be thought of as the virtual machine "source" that virtual machine instances are created from. A "Virtual Machine Instance" is the actual running instance of the virtual machine. The instance is the object you work with that contains IP, networking, and workloads, etc. 

~~~bash
$ oc get vmi
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   77m   Running   192.168.47.34/24   cluster-august-lhrd5-worker-6w624
~~~

In this example we can see the `centos8-server-nfs` instance is on `ocp-9pv98-worker-pj2dn`. As you may recall we deployed this instance with the `LiveMigrate` `evictionStrategy` strategy on an NFS-based, RWX-enabled PVC. You can also review the instance with `oc describe` to ensure it is enabled.

~~~bash
$ oc describe vmi centos8-server-nfs | egrep -i '(eviction|migration)'
        f:evictionStrategy:
        f:migrationMethod:
  Eviction Strategy:  LiveMigrate
  Migration Method:  BlockMigration
~~~

And for the PVC

~~~bash
$ oc describe pvc/centos8-nfs | grep "Access Modes"
Access Modes:  RWO,RWX
~~~

The easiest way to initiate a migration is to create an `VirtualMachineInstanceMigration` object in the cluster directly against the `vmi` we want to migrate. 

**But Wait!**

**Once we create this object it will trigger the migration**, so first, let's just review what it looks like ***without applying it***:

~~~
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: centos8-server-nfs
~~~

It's really quite simple, we create a `VirtualMachineInstanceMigration` object and reference the `LiveMigratable ` instance we want to migrate: `centos8-server-nfs`.  Ok, let's go ahead and apply this configuration:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: centos8-server-nfs
EOF

virtualmachineinstancemigration.kubevirt.io/migration-job created
~~~

Now let's watch the migration job in action. First it will show `phase: Scheduling` 

~~~bash
$ watch "oc get virtualmachineinstancemigration/migration-job -o yaml | tail"

Every 1.0s: oc get virtualmachineinstancemigration/migration-job -o yaml | tail                                                                   Tue Jul 21 20:57:47 2020

    time: "2020-07-24T06:48:34Z"
  name: migration-job
  namespace: default
  resourceVersion: "186751"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Scheduled                           <-----------
~~~

Next `phase: TargetReady`

~~~bash

    time: "2020-07-24T06:48:34Z"
  name: migration-job
  namespace: default
  resourceVersion: "186759"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: TargetReady                          <-----------

~~~

Then `phase: Running `

~~~bash

    time: "2020-07-24T06:48:39Z"
  name: migration-job
  namespace: default
  resourceVersion: "186801"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Running                                 <-----------
~~~

And then to `phase: Succeeded `:

~~~bash

    time: "2020-07-24T06:48:45Z"
  name: migration-job
  namespace: default
  resourceVersion: "186873"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Succeeded                           <-----------
~~~

Pretty quick!

Finally view the `vmi` object and you can see the new underlying host:

~~~bash
$ oc get vmi/centos8-server-nfs
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   81m   Running   192.168.47.34/24   cluster-august-lhrd5-worker-mh52l
~~~

In the above example we have moved the VM from **cluster-august-lhrd5-worker-6w624** to **cluster-august-lhrd5-worker-mh52l** successfully.

Live Migration in OpenShift Virtualization is quite easy. If you have time, try some more migrations. Perhaps start a ping and migrate the machine back. Do you see anything in the ping to indicate the process?

> **NOTE**: If you try and run the same migration job it will report `unchanged`. To run a new job, run the same example as above, but change the job name in the metadata section to something like `name: migration-job2`

For instance:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job2
spec:
  vmiName: centos8-server-nfs
EOF

virtualmachineinstancemigration.kubevirt.io/migration-job2 created
~~~

And so on!

When done with your tests rerun the describe command `oc describe vmi centos8-server-nfs` after running a few migrations. You'll see the object is updated with details of those actions. 

~~~bash
$ oc describe vmi centos8-server-nfs | tail -n 50
  Guest OS Info:
    Id:              centos
    Kernel Release:  4.18.0-193.6.3.el8_2.x86_64
    Kernel Version:  #1 SMP Wed Jun 10 11:09:32 UTC 2020
    Name:            CentOS Linux
    Pretty Name:     CentOS Linux 8 (Core)
    Version:         8
    Version Id:      8
  Interfaces:
    Interface Name:  eth0
    Ip Address:      192.168.47.34/24
    Ip Addresses:
      192.168.47.34/24
      fe80::dcad:beff:feef:1/64
    Mac:             de:ad:be:ef:00:01
    Name:            tuning-bridge-fixed
  Migration Method:  BlockMigration
  Migration State:
    Completed:        true
    End Timestamp:    2020-07-24T06:54:21Z
    Migration UID:    c158e695-6f62-4583-8139-13bf9bc40377
    Source Node:      cluster-august-lhrd5-worker-6w624
    Start Timestamp:  2020-07-24T06:54:11Z
    Target Direct Migration Node Ports:
      34065:                      49152
      37011:                      0
      38249:                      49153
    Target Node:                  cluster-august-lhrd5-worker-mh52l
    Target Node Address:          10.131.0.5
    Target Node Domain Detected:  true
    Target Pod:                   virt-launcher-centos8-server-nfs-t6bvd
  Node Name:                      cluster-august-lhrd5-worker-mh52l
  Phase:                          Running
  Qos Class:                      Burstable
Events:
  Type    Reason            Age                     From                                             Message
  ----    ------            ----                    ----                                             -------
  Normal  SuccessfulCreate  86m                     disruptionbudget-controller                      Created PodDisruptionBudget kubevirt-disruption-budget-ccg67
  Normal  SuccessfulCreate  86m                     virtualmachine-controller                        Created virtual machine pod virt-launcher-centos8-server-nfs-5d8zd
  Normal  Started           86m                     virt-handler, cluster-august-lhrd5-worker-6w624  VirtualMachineInstance started.
  Normal  PreparingTarget   7m34s                   virt-handler, cluster-august-lhrd5-worker-mh52l  Migration Target is listening at 10.131.0.5, on ports: 39687,41135,36627
  Normal  Deleted           7m23s                   virt-handler, cluster-august-lhrd5-worker-6w624  Signaled Deletion
  Normal  PreparingTarget   7m23s (x11 over 7m34s)  virt-handler, cluster-august-lhrd5-worker-mh52l  VirtualMachineInstance Migration Target Prepared.
  Normal  Migrated          7m23s (x2 over 7m23s)   virt-handler, cluster-august-lhrd5-worker-6w624  The VirtualMachineInstance migrated to node cluster-august-lhrd5-worker-mh52l.
  Normal  Migrating         7m23s (x3 over 7m34s)   virt-handler, cluster-august-lhrd5-worker-6w624  VirtualMachineInstance is migrating.
  Normal  Created           3m23s (x13 over 7m23s)  virt-handler, cluster-august-lhrd5-worker-mh52l  VirtualMachineInstance defined.
  Normal  PreparingTarget   2m46s (x2 over 2m46s)   virt-handler, cluster-august-lhrd5-worker-6w624  VirtualMachineInstance Migration Target Prepared.
  Normal  PreparingTarget   2m46s                   virt-handler, cluster-august-lhrd5-worker-6w624  Migration Target is listening at 10.128.2.5, on ports: 45665,41211,45683
  Normal  Created           2m34s (x31 over 86m)    virt-handler, cluster-august-lhrd5-worker-6w624  VirtualMachineInstance defined.
  Normal  Deleted           2m34s                   virt-handler, cluster-august-lhrd5-worker-mh52l  Signaled Deletion
~~~

## Node Maintenance

Building on-top of live migration, many organisations will need to perform node-maintenance, e.g. for software/hardware updates, or for decommissioning. During the lifecycle of a pod, it's almost a given that this will happen without compromising the workloads, but virtual machines can be somewhat more challenging given their legacy nature. Therefore, OpenShift Virtualization has a node-maintenance feature, which can force a machine to no longer be schedulable and any running workloads will be automatically live migrated off if they have the ability to (e.g. using shared storage) and have an appropriate eviction strategy.

Let's take a look at the current running virtual machines and the nodes we have available:

~~~bash
$ oc get nodes
NAME                                STATUS   ROLES    AGE     VERSION
cluster-august-lhrd5-master-0       Ready    master   5h57m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-1       Ready    master   5h57m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-2       Ready    master   5h57m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-6w624   Ready    worker   5h38m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-mh52l   Ready    worker   5h38m   v1.18.3+b74c5ed

$ oc get vmi
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   87m   Running   192.168.47.34/24   cluster-august-lhrd5-worker-mh52
~~~

In this environment, we have one virtual machine instance running on *cluster-august-lhrd5-worker-mh52*. Let's mark that node for maintenance and ensure that our workload (VMI) moves to the available node:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha1
kind: NodeMaintenance
metadata:
  name: worker-maintenance
spec:
  nodeName: cluster-august-lhrd5-worker-mh52l
  reason: "Worker Maintenance - Back Soon"
EOF

nodemaintenance.kubevirt.io/worker-maintenance created
~~~

> **NOTE**: You **may** lose your browser based web terminal, and you'll need to wait a few seconds for it to become accessible again (try refreshing your browser).

Now let's check the status of our environment:

~~~bash
$ oc get nodes
NAME                                STATUS                     ROLES    AGE     VERSION
cluster-august-lhrd5-master-0       Ready                      master   5h58m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-1       Ready                      master   5h58m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-2       Ready                      master   5h58m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-6w624   Ready                      worker   5h39m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-mh52l   Ready,SchedulingDisabled   worker   5h39m   v1.18.3+b74c5ed                   worker   42h   v1.18.3+b74c5ed
~~~

And let's check where our VM went:

~~~bash
$ oc get vmi centos8-server-nfs
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   88m   Running   192.168.47.34/24   cluster-august-lhrd5-worker-6w624
~~~

Success!

Note that the VM has been automatically live migrated to the other worker, as per the `EvictionStrategy`. 

We can remove the maintenance flag by simply deleting the `NodeMaintenance` object:

~~~bash
$ oc get nodemaintenance
NAME                 AGE
worker-maintenance   106s

$ oc delete nodemaintenance/worker-maintenance
nodemaintenance.kubevirt.io "worker-maintenance" deleted
~~~

And the node will be `Ready` again:

~~~bash
$ oc get nodes/cluster-august-lhrd5-worker-mh52l
NAME                                STATUS   ROLES    AGE     VERSION
cluster-august-lhrd5-worker-mh52l   Ready    worker   5h41m   v1.18.3+b74c5ed

$ oc get nodes
NAME                                STATUS   ROLES    AGE     VERSION
cluster-august-lhrd5-master-0       Ready    master   6h      v1.18.3+b74c5ed
cluster-august-lhrd5-master-1       Ready    master   6h      v1.18.3+b74c5ed
cluster-august-lhrd5-master-2       Ready    master   6h      v1.18.3+b74c5ed
cluster-august-lhrd5-worker-6w624   Ready    worker   5h41m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-mh52l   Ready    worker   5h41m   v1.18.3+b74c5ed
~~~

Note the removal of the `SchedulingDisabled` annotation on the '`STATUS` column. 

> **NOTE**: Just because this node has become active again doesn't mean that the virtual machine will 'fail back' and Live Migrate back to it.