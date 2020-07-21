Now let's bring all these settings together and actually launch some workloads!

We are going to launch this VM from a fairly large piece of YAML. Of course, this can also be done from the console, but we will explore that later. 

Let's review what this is going to do.

To begin with let's use the NFS volumes we created earlier to launch some VMs. We are going to create a machine called `centos8-server-nfs`. As you'll recall we have created a PVC called `centos8-nfs` that was created using the CDI utility with a Centos 8 base image. To connect the machine to the network we will utilise the `NetworkAttachmentDefinition` we created for the underlying host's secondary NIC. This is the `tuning-bridge-fixed` interface which refers to that bridge created previously. It's also important to remember that OpenShift 4.x uses Multus as it's default networking CNI so we also ensure Multus knows about this `NetworkAttachmentDefinition`. 

Additionally we have set the evictionStrategy to LiveMigrate so that any request to move the instance will use this method. We will explore this in more depth in a later lab, however note the `ACCESS MODES` set for this machine's PVC: 

~~~bash
[asimonel-redhat.com@bastion cnv]$ oc get pvc
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
centos8-hostpath   Bound    pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8   29Gi       RWO            hostpath-provisioner   39m
centos8-nfs        Bound    nfs-pv1                                    10Gi       RWO,RWX        nfs                    71m
~~~

To enable Live Migration the PVC must be of the type "RWX". RWX stands for "ReadWriteMany". This means that _many_ nodes can mount the volume as _read-write_.

Finally we are using the images built in cloud-init functionality to set the default user's password to "redhat". This means we can login directly to the instance as username centos, password redhat.

Ok, let's create the VM:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/centos8.0: CentOS 8
  name: centos8-server-nfs
  namespace: default
  labels:
    app: centos8-nfs
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/centos8.0: 'true'
    vm.kubevirt.io/template: centos8-server-small-v0.7.0
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.9.1
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/domain: centos8-server-nfs
        kubevirt.io/size: small
        os.template.kubevirt.io/centos8.0: 'true'
        vm.kubevirt.io/name: centos8-server-nfs
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: disk0
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              macAddress: '42:8a:6f:75:d6:4b'
              model: e1000
              name:  tuning-bridge-fixed
          rng: {}
        machine:
          type: pc-q35-rhel8.1.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: centos8-server-nfs
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
        - name: disk0
          persistentVolumeClaim:
            claimName: centos8-nfs
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos8-server-nfs created
~~~

This starts to **schedule** the virtual machine across the available hypervisors, which we can see by viewing the VM and VMI objects:

~~~bash
$ oc get vm
NAME                 AGE   RUNNING   VOLUME
centos8-server-nfs   5s    true

$ oc get vmi
NAME                 AGE   PHASE     IP    NODENAME
centos8-server-nfs   10s   Running         ocp-9pv98-worker-pj2dn
~~~

> **NOTE**: A `vm` object is the definition of the virtual machine, whereas a `vmi` is an instance of that virtual machine definition.

After a few seconds that machine will show as running:

~~~bash
$ oc get vmi
NAME                 AGE     PHASE     IP                NODENAME
centos8-server-nfs   2m18s   Running   192.168.0.28/24   ocp-9pv98-worker-pj2dn
~~~

> **NOTE**: It may take a minute or two for the IP address to appear as it utilise the qemu-guest-agent which needs time to start up.

What you'll find is that OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the `virt-launcher`:

~~~bash
$ oc get pods
NAME                                     READY   STATUS    RESTARTS   AGE
virt-launcher-centos8-server-nfs-58bxn   1/1     Running   0          2m52s

$ oc describe pod/virt-launcher-centos8-server-nfs-58bxn
Name:         virt-launcher-centos8-server-nfs-58bxn
Namespace:    default
Priority:     0
Node:         ocp-9pv98-worker-pj2dn/10.0.0.38
Start Time:   Tue, 21 Jul 2020 00:53:10 -0400
Labels:       flavor.template.kubevirt.io/small=true
              kubevirt.io=virt-launcher
              kubevirt.io/created-by=a239de4d-c54b-44e2-80dd-d2991f87b323
              kubevirt.io/domain=centos8-server-nfs
              kubevirt.io/size=small
              os.template.kubevirt.io/centos8.0=true
              vm.kubevirt.io/name=centos8-server-nfs
              workload.template.kubevirt.io/server=true
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.128.2.9"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "groot",
                    "interface": "net1",
                    "mac": "42:8a:6f:75:d6:4b",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks: [{"interface":"net1","mac":"42:8a:6f:75:d6:4b","name":"tuning-bridge-fixed","namespace":"default"}]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.128.2.9"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "groot",
                    "interface": "net1",
                    "mac": "42:8a:6f:75:d6:4b",
                    "dns": {}
                }]
              kubevirt.io/domain: centos8-server-nfs
Status:       Running
(...)
~~~

If you look into this launcher pod, you'll see that it has the same typical libvirt functionality as we've come to expect with RHV/OpenStack.

First get a shell on the conatiner running pod:

~~~bash
$ oc exec -it virt-launcher-centos8-server-nfs-58bxn /bin/bash

kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
~~~

And then you can run the usual virsh commands:

~~~bash
[root@centos8-server-nfs /]# virsh list --all
 Id   Name                         State
--------------------------------------------
 1    default_centos8-server-nfs   running
~~~

We can also verify the storage attachment, which should be via NFS:

~~~bash
[root@centos8-server-nfs /]# virsh domblklist 1
 Target   Source
----------------------------------------------------------------------------------------------------
 vda      /var/run/kubevirt-private/vmi-disks/disk0/disk.img
 vdb      /var/run/kubevirt-ephemeral-disks/cloud-init-data/default/centos8-server-nfs/noCloud.iso
 
[root@centos8-server-nfs /]# mount | grep nfs
192.168.0.26:/mnt/nfs/one on /run/kubevirt-private/vmi-disks/disk0 type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.0.26,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.0.26)
~~~

And for networking:

~~~bash
[root@centos8-server-nfs /]# virsh domiflist 1
 Interface   Type     Source     Model   MAC
------------------------------------------------------------
 vnet0       bridge   k6t-net1   e1000   42:8a:6f:75:d6:4b
~~~

But let's go a little deeper:

~~~bash
[root@centos8-server-nfs /]# ip link | grep -A2 k6t-net1
5: net1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master k6t-net1 state UP mode DEFAULT group default
    link/ether 42:8a:6f:90:c5:14 brd ff:ff:ff:ff:ff:ff link-netnsid 0
6: k6t-net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 42:8a:6f:90:c5:14 brd ff:ff:ff:ff:ff:ff
7: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether fe:8a:6f:75:d6:4b brd ff:ff:ff:ff:ff:ff
~~~

That's showing that there's a bridge inside of the pod called "**k6t-net1**", with both the **"vnet0"** (the device attached to the VM), and the **"net1@if4"** device being how the packets get out onto the bridge on the hypervisor (more shortly):

~~~bash
[root@centos8-server-nfs /]# virsh dumpxml 1 | grep -A8 "interface type"
    <interface type='bridge'>
      <mac address='42:8a:6f:75:d6:4b'/>
      <source bridge='k6t-net1'/>
      <target dev='vnet0'/>
      <model type='e1000'/>
      <mtu size='1500'/>
      <alias name='ua-tuning-bridge-fixed'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x01' function='0x0'/>
    </interface>
~~~

Exit the shell before proceeding:

~~~bash
[root@rhel8-server-nfs /]# exit
exit

$
~~~

Now, how is this plugged on the underlying host? 

The key to this is the **"net1@if4"** device (it may be slightly different in your environment); this is one half of a **"veth pair"** that allows network traffic to be bridged between network namespaces, which is exactly how containers segregate their network traffic between each other on a container host. In this example the **"cnv-bridge"** is being used to connect the bridge for the virtual machine (**"k6t-net1"**) out to the bridge on the underlying host (**"br1"**), via a veth pair. The other side of the veth pair can be discovered as follows. First find the host of our virtual machine:

~~~bash
$ oc get vmi
NAME                 AGE   PHASE     IP                NODENAME
centos8-server-nfs   10m   Running   192.168.0.28/24   ocp-9pv98-worker-pj2dn
~~~

Then connect to it and track back the link - here you'll need to adjust the commands below - if your veth pair on the pod side was **"net1@if4"** then the **ifindex** in the command below will be **"4"**, if it was **"net1@if5"** then **"ifindex"** will be **"5"** and so on...

To do this we need to get to one of our workers, so first jump to the bastion:

~~~bash
[asimonel-redhat.com@bastion cnv]$ oc debug node/ocp-9pv98-worker-pj2dn
Starting pod/ocp-9pv98-worker-pj2dn-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.38
If you don't see a command prompt, try pressing enter.

sh-4.2# chroot /host

sh-4.4# export ifindex=4

sh-4.4# ip -o link | grep ^$ifindex: | sed -n -e 's/.*\(veth[[:alnum:]]*@if[[:digit:]]*\).*/\1/p'
veth9aafe530@if5
~~~

Therefore, the other side of the link, in the example above is **"veth9aafe530"**. You can then see that this is attached to **"br1"** as follows-

~~~bash
sh-4.4# ip link show dev veth9aafe530
4: veth9aafe530@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br1 state UP mode DEFAULT group default
    link/ether 22:56:9d:ef:cf:d9 brd ff:ff:ff:ff:ff:ff link-netns ecd9d185-382c-4252-a9b8-2d110498ffc0
~~~

Note the "**master br1**" in the above output.

Or visually represented:

<img src="img/veth-pair.png" />

Exit the debug shell(s) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv

$ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~

Now that we have the NFS instance running, let's do the same for the **hostpath** setup we created. This is essentially the same as our NFS instance, except we reference the `rhel8-hostpath` PVC:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/centos8.0: CentOS 8
  name: centos8-server-hostpath
  namespace: default
  labels:
    app: centos8-server-hostpath
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/centos8.0: 'true'
    vm.kubevirt.io/template: centos8-server-small-v0.7.0
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.9.1
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/domain: centos8-server-hostpath
        kubevirt.io/size: small
        os.template.kubevirt.io/centos8.0: 'true'
        vm.kubevirt.io/name: centos8-server-hostpath
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: sata
              name: centos8-hostpath
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              macAddress: '42:8a:6f:75:d6:4c'
              model: e1000
              name:  tuning-bridge-fixed
          rng: {}
        machine:
          type: pc-q35-rhel8.1.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: centos8-server-hostpath
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
        - name: centos8-hostpath
          persistentVolumeClaim:
            claimName: centos8-hostpath
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              password: redhat
              chpasswd: {expire: False}
          name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos8-server-hostpath created
~~~

As before we can see the launcher pod built and run:

~~~bash
$ oc get pods
NAME                                          READY   STATUS    RESTARTS   AGE
virt-launcher-centos8-server-hostpath-5mmxw   1/1     Running   0          12s
virt-launcher-centos8-server-nfs-58bxn        1/1     Running   0          18m

$ oc get vmi
NAME                      AGE     PHASE     IP                NODENAME
centos8-server-hostpath   2m18s   Running   192.168.0.41/24   ocp-9pv98-worker-pj2dn
centos8-server-nfs        20m     Running   192.168.0.28/24   ocp-9pv98-worker-pj2dn

~~~

And looking deeper we can see the hostpath claim we explored earlier being utilised, note the `Mounts` section for where, inside the pod, the `rhel8-hostpath` PVC is attached, and then below the PVC name:

~~~bash
$ oc describe pod/virt-launcher-centos8-server-hostpath-5mmxw
Name:         virt-launcher-centos8-server-hostpath-5mmxw
Namespace:    default
Priority:     0
Node:         ocp-9pv98-worker-pj2dn/10.0.0.38
Start Time:   Tue, 21 Jul 2020 01:11:07 -0400
Labels:       flavor.template.kubevirt.io/small=true
              kubevirt.io=virt-launcher
              kubevirt.io/created-by=8cfc54a6-ee75-4e07-a789-137ba8fb763a
              kubevirt.io/domain=centos8-server-hostpath
              kubevirt.io/size=small
              os.template.kubevirt.io/centos8.0=true
              vm.kubevirt.io/name=centos8-server-hostpath
(...)
    Mounts:
      /var/run/kubevirt from virt-share-dir (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-infra from infra-ready-mount (rw)
      /var/run/kubevirt-private/vmi-disks/centos8-hostpath from centos8-hostpath (rw)
      /var/run/kubevirt/container-disks from container-disks (rw)
      /var/run/libvirt from libvirt-runtime (rw)

(...)
Volumes:
  centos8-hostpath:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  centos8-hostpath
    ReadOnly:   false
(...)
~~~

If we take a peek inside of that pod we can dig down further, you'll notice that it's largely the same output as the NFS step above, but the mount point is obviously not over NFS:

~~~bash
$ oc exec -it virt-launcher-centos8-server-hostpath-5mmxw /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead

[root@centos8-server-hostpath /]#  virsh domblklist 1
 Target   Source
---------------------------------------------------------------------------------------------------------
 sda      /var/run/kubevirt-private/vmi-disks/centos8-hostpath/disk.img
 vda      /var/run/kubevirt-ephemeral-disks/cloud-init-data/default/centos8-server-hostpath/noCloud.iso

[root@centos8-server-hostpath /]# mount | grep centos8
/dev/mapper/coreos-luks-root-nocrypt on /run/kubevirt-private/vmi-disks/centos8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,prjquota)
~~~

The problem here is that if you try and run the `mount` command to see how this is attached, it will only show the whole filesystem being mounted into the pod:

~~~bash
[root@centos8-server-hostpath /]# mount | grep centos8
/dev/mapper/coreos-luks-root-nocrypt on /run/kubevirt-private/vmi-disks/centos8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,prjquota)
~~~

However, we can see how this has been mapped in by Kubernetes by looking at the pod configuration on the host. First, recall which host your virtual machine is running on, and get the name of the launcher pod:

~~~bash
[root@rhel8-server-hostpath /]# exit
logout

$ oc get vmi/centos8-server-hostpath
NAME                      AGE     PHASE     IP                NODENAME
centos8-server-hostpath   4m29s   Running   192.168.0.41/24   ocp-9pv98-worker-pj2d

$ oc get pods | grep centos8-server-hostpath
virt-launcher-centos8-server-hostpath-5mmxw   1/1     Running   0          4m54s
~~~

Next, we need to get the container ID from the pod:

~~~bash
$ oc describe pod virt-launcher-centos8-server-hostpath-5mmxw | awk -F// '/Container ID/ {print $2;}'
9823cde09cfd31b4d0d1273892d8e22120accec0bcdfc579bc25434fbf6d0f08
~~~

Now we can check on the worker node itself, remembering to adjust these commands for the worker that your hostpath based VM is running on, and the container ID from the above:

~~~bash
[asimonel-redhat.com@bastion cnv]$ oc debug node/ocp-9pv98-worker-pj2dn
Starting pod/ocp-9pv98-worker-pj2dn-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.38
If you don't see a command prompt, try pressing enter.

sh-4.2# chroot /host

sh-4.4# crictl inspect 9823cde09cfd31b4d0d1273892d8e22120accec0bcdfc579bc25434fbf6d0f08 | grep -A4 centos8-hostpath
        "containerPath": "/var/run/kubevirt-private/vmi-disks/centos8-hostpath",
        "hostPath": "/var/hpvolumes/pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": false
(...)
~~~

Here you can see that the container has been configured to have a `hostPath` from `/var/hpvolumes` mapped to the expected path inside of the container where the libvirt definition is pointing to.

Don't forget to exit (twice) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...

$ oc whoami
system:serviceaccount:workbook:cnv

[~] $ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~


