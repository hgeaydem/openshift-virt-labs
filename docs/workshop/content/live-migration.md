Live Migration is the process of moving an instance from one node in a cluster to another without interruption. This process can be manual or automatic. This depends if the `evictionStrategy` strategy is set to `LiveMigrate` and the underlying node is placed into maintenance. It also depends on the storage access mode. Virtual machines must have a PersistentVolumeClaim (PVC) with a shared ReadWriteMany (RWX) access mode to be live migrated.

Live migration is an administrative function in OpenShift Virtualization. While the action is visible to all users, only admins can initiate a migration. Migration limits and timeouts are managed via the `kubevirt-config` `configmap`. For more details about limits see the [documentation](https://docs.openshift.com/container-platform/4.4/cnv/cnv_live_migration/cnv-live-migration-limits.html#cnv-live-migration-limits).

In our lab you should now have only one VM running. You can check that, and view the underlying host it is on, by looking at the virtual machine's instance with the `oc get vmi` command.

> **NOTE**: In OpenShift Virtualization, the "Virtual Machine" object can be thought of as the virtual machine "source" that virtual machine instances are created from. A "Virtual Machine Instance" is the actual running instance of the virtual machine. The instance is the object you work with that contains IP, networking, and workloads, etc. 

~~~bash
$ oc get vmi
NAME                 AGE   PHASE     IP                NODENAME
centos8-server-nfs   19h   Running   192.168.0.28/24   ocp-9pv98-worker-pj2dn
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

    time: "2020-07-22T00:55:42Z"
  name: migration-job
  namespace: default
  resourceVersion: "1206211"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: 3a06db36-7a07-4819-84c4-f056d0b406eb
spec:
  vmiName: centos8-server-nfs
status:
  phase: Scheduling                                 <-----------
~~~

Then `Running`

~~~bash

    time: "2020-07-22T00:55:42Z"
  name: migration-job
  namespace: default
  resourceVersion: "1207123"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: 3a06db36-7a07-4819-84c4-f056d0b406eb
spec:
  vmiName: centos8-server-nfs
status:
  phase: Running                                 <-----------
~~~

And then to `Succeeded`:

~~~bash

    time: "2020-07-22T00:55:42Z"
  name: migration-job
  namespace: default
  resourceVersion: "1208521"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job
  uid: 3a06db36-7a07-4819-84c4-f056d0b406eb
spec:
  vmiName: centos8-server-nfs
status:
  phase: Succeeded                             <-----------
~~~

Finally view the `vmi` object and you can see the new underlying host:

~~~bash
$ oc get vmi/centos8-server-nfs
NAME                 AGE   PHASE     IP                NODENAME
centos8-server-nfs   20h   Running   192.168.0.28/24   ocp-9pv98-worker-g78bj
~~~

In the above example we have moved the VM from **ocp-9pv98-worker-pj2dn** to **ocp-9pv98-worker-g78bj** successfully.

Live Migration in OpenShift virtualisation is quite easy. If you have time, try some other examples. Perhaps start a ping and migrate the machine back. Do you see anything in the ping to indicate the process?

> **NOTE**: If you try and run the same migration job it will report `unchanged`. To run a new job, run the same example as above, but change the job name in the metadata section to something like `name: migration-job2`

Also, rerun the `oc describe vmi centos8-server-nfs` after running a few migrations. You'll see the object is updated with details of the migrations:

~~~bash
$ oc describe vmi centos8-server-nfs
(...)
  Migration Method:  BlockMigration
  Migration State:
    Completed:        true
    End Timestamp:    2020-07-22T01:02:38Z
    Migration UID:    44885086-9e79-4d34-9de8-799184ae5818
    Source Node:      ocp-9pv98-worker-pj2dn
    Start Timestamp:  2020-07-22T01:02:31Z
    Target Direct Migration Node Ports:
      38725:                      49152
      43441:                      0
      46869:                      49153
    Target Node:                  ocp-9pv98-worker-g78bj
    Target Node Address:          10.131.0.4
    Target Node Domain Detected:  true
    Target Pod:                   virt-launcher-centos8-server-nfs-kkp2m
  Node Name:                      ocp-9pv98-worker-g78bj
  Phase:                          Running
  Qos Class:                      Burstable
Events:
  Type    Reason           Age                     From                                  Message
  ----    ------           ----                    ----                                  -------
  Normal  PreparingTarget  9m56s                   virt-handler, ocp-9pv98-worker-g78bj  Migration Target is listening at 10.131.0.4, on ports: 33213,33617,41767
  Normal  PreparingTarget  9m47s (x11 over 9m56s)  virt-handler, ocp-9pv98-worker-g78bj  VirtualMachineInstance Migration Target Prepared.
  Normal  Deleted          9m47s                   virt-handler, ocp-9pv98-worker-pj2dn  Signaled Deletion
  Normal  Created          4m48s (x14 over 9m47s)  virt-handler, ocp-9pv98-worker-g78bj  VirtualMachineInstance defined.
  Normal  PreparingTarget  3m49s                   virt-handler, ocp-9pv98-worker-pj2dn  Migration Target is listening at 10.128.2.4, on ports: 35217,38319,37603
  Normal  PreparingTarget  3m48s (x2 over 3m49s)   virt-handler, ocp-9pv98-worker-pj2dn  VirtualMachineInstance Migration Target Prepared.
  Normal  Created          2m58s (x38 over 20h)    virt-handler, ocp-9pv98-worker-pj2dn  VirtualMachineInstance defined.
  Normal  Migrating        2m51s (x12 over 9m56s)  virt-handler, ocp-9pv98-worker-pj2dn  VirtualMachineInstance is migrating.
  Normal  Migrated         2m51s (x4 over 9m47s)   virt-handler, ocp-9pv98-worker-pj2dn  The VirtualMachineInstance migrated to node ocp-9pv98-worker-g78bj.
~~~

## Node Maintenance

Building on-top of live migration, many organisations will need to perform node-maintenance, e.g. for software/hardware updates, or for decommissioning. During the lifecycle of a pod, it's almost a given that this will happen without compromising the workloads, but virtual machines can be somewhat more challenging given their legacy nature. Therefore, OpenShift Virtualization has a node-maintenance feature, which can force a machine to no longer be schedulable and any running workloads will be automatically live migrated off if they have the ability to (e.g. using shared storage) and have an appropriate eviction strategy.

Let's take a look at the current running virtual machines and the nodes we have available:

~~~bash
$ oc get nodes
NAME                     STATUS                     ROLES    AGE   VERSION
ocp-9pv98-master-0       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-master-1       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-master-2       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-worker-g78bj   Ready                      worker   42h   v1.18.3+b74c5ed
ocp-9pv98-worker-pj2dn   Ready                      worker   42h   v1.18.3+b74c5ed

$ oc get vmi
NAME                 AGE   PHASE     IP                NODENAME
centos8-server-nfs   20h   Running   192.168.0.28/24   ocp-9pv98-worker-g78bj
~~~

In this environment, we have one virtual machine running on *ocp-9pv98-worker-g78bj*. Let's take down the node for maintenance and ensure that our workload (VM) stays up and running:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha1
kind: NodeMaintenance
metadata:
  name: worker-maintenance
spec:
  nodeName: ocp-9pv98-worker-g78bj
  reason: "Worker Maintenance - Back Soon"
EOF

nodemaintenance.kubevirt.io/worker-maintenance created
~~~

> **NOTE**: You **may** lose your browser based web terminal, and you'll need to wait a few seconds for it to become accessible again (try refreshing your browser).

Now let's check the status of our environment:

~~~bash
$ oc project default
Now using project "default" on server "https://172.30.0.1:443".

$ oc get nodes
NAME                     STATUS                     ROLES    AGE   VERSION
ocp-9pv98-master-0       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-master-1       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-master-2       Ready                      master   43h   v1.18.3+b74c5ed
ocp-9pv98-worker-g78bj   Ready,SchedulingDisabled   worker   42h   v1.18.3+b74c5ed
ocp-9pv98-worker-pj2dn   Ready                      worker   42h   v1.18.3+b74c5ed
~~~

And let's check where our VM went:

~~~bash
$ oc get vmi centos8-server-nfs
NAME                 AGE   PHASE     IP                NODENAME
centos8-server-nfs   20h   Running   192.168.0.28/24   ocp-9pv98-worker-pj2dn
~~~

Back to **ocp-9pv98-worker-pj2dn**!

Note that the VM has been automatically live migrated back to the other worker, as per the `EvictionStrategy`. 

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
$ oc get nodes/ocp-9pv98-worker-g78bj
NAME                     STATUS   ROLES    AGE   VERSION
ocp-9pv98-worker-g78bj   Ready    worker   42h   v1.18.3+b74c5ed
~~~

Note the removal of the `SchedulingDisabled` annotation on the '`STATUS` column. 

Be advised that just because this node has become active again it doesn't mean that the virtual machine will 'fail back' and Live Migrate back to it.