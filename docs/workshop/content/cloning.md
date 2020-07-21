In this lab we're going to clone a workload and prove that it's identical to the previous. For fun, we will use a Centos 7 image for this work. We're going to download and customise the image, launch a virtual machine via OpenShift Virtualization based on it, and then clone it - we'll then test to see if the cloned machine works as expected. 

Let's begin by checking we have a availale PV for this work:

~~~bash
[asimonel-redhat.com@bastion cnv]$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                             91m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                             78m
pvc-2c361fda-f643-45bb-8c53-d714ff2cfa0f   100Gi      RWO            Delete           Bound       openshift-image-registry/image-registry-storage   standard                        23h
pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8   29Gi       RWO            Delete           Bound       default/centos8-hostpath                          hostpath-provisioner            3h1m
~~~

You should have nfs-pv2 marked as `Available`.


And make sure the volume is `Available`:

Next we will create a PVC for that PV that utilises the CDI utility to import the Centos 7 image (from http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2). The syntax should be familiar to you now with the `cdi.kubevirt.io/storage.import.endpoint` annotation indicating the endpoint for CDI to import from.

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos7-clone-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/centos7-clone-nfs created
~~~

As before we can watch the process and see the pods:

~~~bash
$ oc get pods
NAME                                          READY   STATUS    RESTARTS   AGE
importer-centos7-clone-nfs                    1/1     Running   0          18s
virt-launcher-centos8-server-hostpath-5mmxw   1/1     Running   0          24m
virt-launcher-centos8-server-nfs-58bxn        1/1     Running   0          42m
~~~

Be fast!

~~~bash
$ oc logs -f importer-centos7-clone-nfs
I0721 05:54:57.926925       1 importer.go:51] Starting importer
I0721 05:54:57.927727       1 importer.go:107] begin import process
I0721 05:54:58.090877       1 data-processor.go:275] Calculating available size
I0721 05:54:58.094471       1 data-processor.go:283] Checking out file system volume size.
I0721 05:54:58.094909       1 data-processor.go:287] Request image size not empty.
I0721 05:54:58.094940       1 data-processor.go:292] Target size 10Gi.
I0721 05:54:58.096037       1 util.go:37] deleting file: /data/disk.img
I0721 05:54:58.208590       1 data-processor.go:205] New phase: Convert
I0721 05:54:58.208617       1 data-processor.go:211] Validating image
I0721 05:54:58.391211       1 qemu.go:212] 0.00
I0721 05:54:58.832161       1 qemu.go:212] 1.23
I0721 05:54:59.167900       1 qemu.go:212] 2.36
I0721 05:54:59.311628       1 qemu.go:212] 3.58
...
I0721 05:55:12.209566       1 qemu.go:212] 96.81
I0721 05:55:12.669545       1 qemu.go:212] 98.00
I0721 05:55:12.719049       1 qemu.go:212] 99.22
I0721 05:55:13.098490       1 data-processor.go:205] New phase: Resize
I0721 05:55:13.112280       1 data-processor.go:268] Expanding image size to: 10Gi
I0721 05:55:13.128083       1 data-processor.go:205] New phase: Complete
I0721 05:55:13.128221       1 importer.go:160] Import complete
~~~

The bound PV:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound    default/centos8-nfs                               nfs                             96m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Bound    default/centos7-clone-nfs                         nfs                             83m
pvc-2c361fda-f643-45bb-8c53-d714ff2cfa0f   100Gi      RWO            Delete           Bound    openshift-image-registry/image-registry-storage   standard                        23h
pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8   29Gi       RWO            Delete           Bound    default/centos8-hostpath                          hostpath-provisioner
~~~

And finally our working PVC:

~~~bash
$ oc get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
centos7-clone-nfs   Bound    nfs-pv2                                    10Gi       RWO,RWX        nfs                    73s
centos8-hostpath    Bound    pvc-cf71987e-a9ff-4f2a-85de-124c578acaf8   29Gi       RWO            hostpath-provisioner   3h6m
centos8-nfs         Bound    nfs-pv1                                    10Gi       RWO,RWX        nfs                    77m
~~~

### Centos 7 Virtual Machine

Now it's time to launch our Centos 7 VM. Again we are just using the same pieces we've been using throughout the labs. For review we are using the `centos7-clone-nfs` PVC we just prepared (created with CDI importing the Centos image, stored on NFS), and we are utilising the standard bridged networking on the workers via the `tuning-bridge-fixed` construct - the same as we've been using for the other two virtual machines.

However we are going to run quite a bit more via cloud-init. This time we will do the following:

* set the centos user password to "redhat"
* enable ssh logins
* install podman
* set up an ngnix podman container to allow us to access a simple web page on the host

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
 annotations:
   kubevirt.io/latest-observed-api-version: v1alpha3
   kubevirt.io/storage-observed-api-version: v1alpha3
   name.os.template.kubevirt.io/centos7.0: CentOS 7
 name: centos7-clone-nfs
 namespace: default
 labels:
   app: centos7-clone-nfs
   flavor.template.kubevirt.io/small: 'true'
   os.template.kubevirt.io/centos7.0: 'true'
   vm.kubevirt.io/template: centos7-server-small-v0.7.0
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
       kubevirt.io/domain: centos7-clone-nfs
       kubevirt.io/size: small
       os.template.kubevirt.io/centos7.0: 'true'
       vm.kubevirt.io/name: centos7-clone-nfs
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
             macAddress: '42:8a:6f:75:d6:4d'
             model: e1000
             name:  tuning-bridge-fixed
         rng: {}
       machine:
         type: pc-q35-rhel8.1.0
       resources:
         requests:
           memory: 2Gi
     evictionStrategy: LiveMigrate
     hostname: centos7-clone-nfs
     networks:
       - multus:
           networkName: tuning-bridge-fixed
         name: tuning-bridge-fixed
     terminationGracePeriodSeconds: 0
     volumes:
       - name: disk0
         persistentVolumeClaim:
           claimName: centos7-clone-nfs-fresh
       - cloudInitNoCloud:
           userData: |-
             #cloud-config
             password: redhat
             chpasswd: {expire: False}
             ssh_pwauth: 1
             packages:
               - podman
             runcmd:
               - [ systemctl, daemon-reload ]
               - [ systemctl, enable, nginx ]
               - [ systemctl, start, --no-block, nginx ]
             write_files:
               - path: /etc/systemd/system/nginx.service
                 permissions: ‘0755’
                 content: |
                     [Unit]
                     Description=Nginx Podman container
                     Wants=syslog.service
                     [Service]
                     ExecStart=/usr/bin/podman run --net=host nginxdemos/hello
                     ExecStop=/usr/bin/podman stop --all
                     [Install]
                     WantedBy=multi-user.target
         name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos7-clone-nfs created
~~~

We can view the running VM:

~~~bash
AUGUST
~~~

ssh should work

~~~bash
$ ssh root@192.168.

AUGUST HERE
~~~


Let's install `nginx` via `systemd` and `podman`, i.e. have *systemd* call *podman* to start an *nginx* container at boot time:

~~~bash
[root@localhost ~]# dnf install podman -y


Last metadata expiration check: 0:38:21 ago on Thu 19 Mar 2020 03:42:07 AM UTC.
Dependencies resolved.
==================================================================================================================================================================================
 Package                                             Architecture                   Version                                                 Repository                       Size
==================================================================================================================================================================================
Installing:
 podman                                              x86_64                         2:1.8.1-2.fc31                                          updates                          13 M
(...)

[root@localhost ~]# cat >> /etc/systemd/system/nginx.service << EOF
[Unit]
Description=Nginx Podman container
Wants=syslog.service
[Service]
ExecStart=/usr/bin/podman run --net=host nginxdemos/hello
ExecStop=/usr/bin/podman stop --all
[Install]
WantedBy=multi-user.target
EOF

[root@localhost ~]# systemctl enable --now nginx

Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /etc/systemd/system/nginx.service.

[root@localhost ~]# systemctl status nginx
● nginx.service - Nginx Podman container
   Loaded: loaded (/etc/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2020-03-31 01:30:48 UTC; 8s ago
 Main PID: 9898 (podman)
    Tasks: 11 (limit: 2345)

(should see "active (running)" in green above)

[root@localhost ~]# logout
Connection to 192.168.123.64 closed.

$
~~~

Let's quickly verify that this works as expected - you should be able to navigate directly to the IP address of your machine in your browser - recalling that in my example it's *192.168.123.64*, it may be different for you, but unlikely if you've not created any additional VM's along the way:

<img src="img/nginx-fc31.png"/>

> **NOTE**: These steps are important for both this lab and a future one; please ensure they complete correctly.

We need to shutdown the VM so we can clone it without risking filesystem corruption, but for us to do that we need to change some of the default behaviour of our virtual machine to stop OpenShift and OpenShift virtualisation from automatically restarting our VM if we do try to shut it down. We need to remove the `running: true` line, and replace it with `runStrategy: RerunOnFailure` within the the `vm` object:

~~~bash
$ oc edit vm/fc31-nfs
(...)

1. Remove the line containing "running: true"
2. Replace it with "runStrategy: RerunOnFailure"
3. Save the file and exit the editor

virtualmachine.kubevirt.io/fc31-nfs edited
~~~

> **NOTE**: You can prefix the `oc edit` command with "`EDITOR=<your favourite editor>`" if you'd prefer to use something other than vi to do this work. Also note that "ReRunOnFailure" simply means that OpenShift virtualisation will do its best to automatically restart this VM if a failure has occurred, but won't always enforce it to be running, e.g. if the user shuts it down


Now reopen the  ssh-session to the Fedora 31 machine and power it off:

~~~bash
$ oc get vmi/fc31-nfs
NAME       AGE   PHASE     IP                  NODENAME
fc31-nfs   18m   Running   192.168.123.64/24   ocp4-worker1.cnv.example.com

$ ssh root@192.168.123.64
(password is "redhat")

[root@localhost ~]# poweroff
[root@localhost ~]# Connection to 192.168.123.64 closed by remote host.
Connection to 192.168.123.64 closed.
~~~

Now if you check the list of `vmi` objects you should see that it's marked as `Succeeded` rather than being automatically restarted by OpenShift virtualisation:

~~~bash
$ oc get vmi
NAME                    AGE   PHASE       IP                  NODENAME
fc31-nfs                19m   Succeeded   192.168.123.65/24   ocp4-worker1.cnv.example.com
rhel8-server-hostpath   91m   Running     192.168.123.63/24   ocp4-worker1.cnv.example.com
rhel8-server-nfs        94m   Running     192.168.123.62/24   ocp4-worker1.cnv.example.com
~~~



### Clone the VM

Now that we've got a working virtual machine with a test workload we're ready to actually clone it, to prove that the built-in cloning utilities work, and that the cloned machine shares the same workload. First we need to create a PV (persistent volume) to clone into. This is done by creating a special resource called a `DataVolume`, this custom resource type is provide by CDI. DataVolumes orchestrate import, clone, and upload operations and help the process of importing data into a cluster. DataVolumes are integrated into OpenShift virtualisation.

The volume we are creating is named `fc31-clone`, we'll be pulling the data from the volume ("source") `fc31-nfs` and we'll tell the CDI to provision onto a specific node. We're only using this option to demonstrate the annotation, but also because we're going to clone from an NFS-based volume onto a hostpath based volume; this is important because hostpath volumes are not shared-storage based, so the location you specify here is important as that's exactly where the cloned VM will have to run:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: cdi.kubevirt.io/v1alpha1
kind: DataVolume
metadata:
  name: fc31-clone
spec:
  source:
    pvc:
      namespace: default
      name: fc31-nfs
  pvc:
    accessModes:
      - ReadWriteOnce
    storageClassName: hostpath-provisioner
    resources:
      requests:
        storage: 20Gi
EOF

datavolume.cdi.kubevirt.io/fc31-clone created
~~~

You can watch the progress with where it will go through `CloneScheduled` and `CloneInProgress` phases along with a handy status precentage:

~~~bash
$ watch -n5 oc get datavolume
Every 5.0s: oc get datavolume

NAME         PHASE            PROGRESS   AGE
fc31-clone   CloneScheduled              39s
(...)

NAME         PHASE             PROGRESS   AGE
fc31-clone   CloneInProgress   27.38%     2m46s
(...)

NAME         PHASE       PROGRESS   AGE
fc31-clone   Succeeded   100.0%     3m13s

(Ctrl-C to stop/quit)
~~~

> **NOTE**: It may take a few minutes for the clone to start as it's got to pull the CDI clone images down too.



View all your PVCs, and the new clone:

~~~bash
$ oc get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
fc31-clone       Bound    pvc-64de80bd-0805-45bc-81d3-56482e609705   79Gi       RWO            hostpath-provisioner   5m10s
fc31-nfs         Bound    fc31-pv                                    10Gi       RWO,RWX        nfs                    75m
rhel8-hostpath   Bound    pvc-6f7eb369-6605-486c-93b9-07d8069aad34   79Gi       RWO            hostpath-provisioner   106m
rhel8-nfs        Bound    nfs-pv1                                    40Gi       RWO,RWX        nfs                    120m11
~~~



### Start the cloned VM

Finally we can start up a new VM using the cloned PVC:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: fc31-clone
  labels:
    app: fc31-clone
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/fedora31: 'true'
    vm.kubevirt.io/template: fedora-server-small-v0.7.0
    vm.kubevirt.io/template-namespace: openshift
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      labels:
        flavor.template.kubevirt.io/small: 'true'
        kubevirt.io/size: small
        os.template.kubevirt.io/fedora31: 'true'
        vm.kubevirt.io/name: fc31-clone
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          autoattachPodInterface: false
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: disk0
          interfaces:
            - bridge: {}
              model: virtio
              name: nic0
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel8.1.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: fc31-clone
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: nic0
      terminationGracePeriodSeconds: 0
      volumes:
        - name: disk0
          persistentVolumeClaim:
            claimName: fc31-clone
EOF

virtualmachine.kubevirt.io/fc31-clone created
~~~

After a few minutes you should see the new virtual machine running:

~~~bash
$ oc get vmi
NAME                    AGE     PHASE       IP                  NODENAME
fc31-clone              6m15s   Running     192.168.123.67/24   ocp4-worker2.cnv.example.com
fc31-nfs                37m     Succeeded   192.168.123.64/24   ocp4-worker1.cnv.example.com
rhel8-server-hostpath   109m    Running     192.168.123.63/24   ocp4-worker1.cnv.example.com
rhel8-server-nfs        111m    Running     192.168.123.62/24   ocp4-worker1.cnv.example.com
~~~

> **Note** Give the command 2-3 minutes to report the IP. 

This machine will also be visible from the console, and you can login using "**root/redhat**" just like before:

<img src="img/fc31-clone-console.png"/>

### Test the clone

Connect your browser to http://192.168.123.67/ (noting that your IP from the clone and that your IP may be different) and you should find the **ngnix** server you configured on the original host, prior to the clone, pleasantly serving your request:

<img src="img/ngnix-clone.png"/>



That's it! You've proven that your clone has worked, and that the hostpath based volume is an identical copy of the original NFS-based one.

### Clean up

Before moving on to the next lab let's clean up the VMs so we ensure our environment has all the resources it might need; we're going to delete our Fedora VMs and our RHEL8 hostpath VM:

~~~bash
$ oc delete vm/fc31-clone vm/fc31-nfs vm/rhel8-server-hostpath
virtualmachine.kubevirt.io "fc31-clone" deleted
virtualmachine.kubevirt.io "fc31-nfs" deleted
virtualmachine.kubevirt.io "rhel8-server-hostpath" deleted
~~~

> **NOTE**: If you check `oc get vmi` too quickly you might see the VMs in a `Failed` state. This is normal and when you check again they should disappear accordingly.

~~~bash
$ oc get vmi
NAME               AGE   PHASE     IP                  NODENAME
rhel8-server-nfs   23h   Running   192.168.123.62/24   ocp4-worker1.cnv.example.com
~~~

