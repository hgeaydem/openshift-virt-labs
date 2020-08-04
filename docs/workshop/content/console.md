Now that you've had a chance to dive into how OpenShift virtualisation works at a very deep level, let's spend a little time looking at an equally important component of the OpenShift virtualisation experience: the User Interface (UI). As with all Red Hat products OpenShift virtualisation offers a rich CLI allowing administrators to easily script repetitive actions, dive deep into components, and create almost infinite levels of configuration. In these labs you've use both `virtctl`, the OpenShift virtualisation client, and `oc`, the OpenShift Container Platform client, to do many tasks.

However, the need for an easy to use, developer friendly User Experience is also a key value for OpenShift and OpenShift virtualisation. Let's take a look at some of the outcomes and actions from the previous labs and how they can be achived and/or reviewed from within OpenShift's exceptional UI.

## Connecting to the console
As you know you can view the OpenShift console (its User Interface, UI) by clicking on the "Console" link in the lab's environment:

<img src="img/console-button.png"/>

This brings up the familiar OpenShift console with the lab text to the side. You may need to adjust the panes to ensure you can see all the panels.

<img src="img/ui/ui-1.png"/>

## Virtual Machines in OpenShift!

Once OpenShift virtualisation is installed an option called **"Virtualization"** appeara under the **"Workloads"** section. And once you have running VM's they are added for easy access from the **"Cluster Inventory"** Panel:

<img src="img/ui/ui-2.png"/>

Choose "**Virtualization**" from the "**Workloads**" menu and you will see the running VMs from the labs. Click on the link for the "**centos8-server-nfs**" instance. You can see some of the familiar features of this instance, such as the IP, node, OS, and more.

<img src="img/ui/ui-3.png"/>

Across the top is a helpful menu allowing you deeper access to administer and access the running VM. 

* Choose `YAML` to view, edit, and reload the template defining this instance. 
* `Consoles` give you access, via both VNC and Serial to the instance's console for easy login. 
* `Events` display, of course, important events for the instance. 
* With `Network Interfaces` we can see the name, model, binding, etc for the instances network connections. 
* Under `Disks` we can see the storage class used and the type of interface. 

Be sure to look at the different entries for different types of virtual machines. You should be able to directly connect the names, values, and information to the services and objects you created before.

But it's not just the VM instances available. OpenShift virtualisation is merely an extension to OpenShift and all the components we created and used for our VMs are part of the OpenShift experience just like they are for pods. On the menu on the left choose "**Networking**" and select "**Services**":

<img src="img/ui/ui-4.png"/>

Here you can see the networking services we created for our Centos 7 host for **http**, **ssh**, and  **NodePort SSH**. Try drilling down on the `centos7-masq-externalport` Service. You should see all the features of the service  including Cluster IP, port mapping, and routing. From the actions menu in the upper right corner you can even edit these values directly from this UI to update your VM's service directly from the UI, in the same was as you do with OpenShift pods.

<img src="img/ui/ui-5.png"/>

Next click on the `Pods` menu item.

Here we can see the pod that is running the VM instance (the "launcher" pod we learned about previously) as well as the host it is running on:

<img src="img/ui/ui-6.png"/>

Choose the link for the launcher pod and we can really see the components we built behind the pod. It's important again to note that from here we have access like any other pod in OpenShift. We can use the menu across the top to view the Pod's YAML, Logs, and even access the pod's Terminal (note this is NOT the VM's command line or console, this is for the pod which **runs** the VM; remember when we did this from the CLI?!).

<img src="img/ui/ui-7.png"/>

Go back to the *Details* tab and scroll down on this screen to the Volumes section. Here we can see all the usual pod constructs, such as volumes and disks, but we also see the mounted disk for the VM it's running: `disk-0`. 

<img src="img/ui/ui-8.png"/>

Following on from here, select the link for `disk-0` under the PVC label: 

<img src="img/ui/ui-9.png"/>

Here we are able to see and edit all the features of the PVC we assigned for this pod.

<img src="img/ui/ui-10.png"/>

Now, under the **"Storage"** menu on the left, choose "**Persistent Volumes**" and you'll see the storage setup we created previously.

<img src="img/ui/ui-11.png"/>

The same is true for all components of our VM's and VMI's from the labs. If you choose any of the storage items from the now-highlighted storage menu on the left you will see. Try choosing "**Storage**" > "**Persistent Volumes**" to show the various PVs we created. Continue to "**Persistent Volume Claims**" and "**Storage Classes**" to complete the picture.

Finally, select "**Networking**" and choose "**Network Attachment Definitions**". There you'll see the `tuning-bridge-fixed` policy we created. As you can see from this page you can even use the UI to crate these definitions!

<img src="img/ui/ui-12.png"/>


## Preparing to Launch a VM

Now let's launch a Centos VM via the UI using the same components we used via the CLI. 

#### Storage

First, let's create our storage. As before let's create a PVC using CDI to import our OS. However, as you've just seen through our review steps above we do not have any unbound PV's. So let's use the UI to create one.

Go to **"Storage"** -> "**Persistent Volumes**" and choose the "**Create Persistent Volume**" button in the upper right corner:

<img src="img/ui/ui-13.png"/>

Here there is no fancy wizard to create what we need, so we have to supply the YAML directly. We have an existing, unused NFS share on our bastion host under the /mnt/nfs/three directory. So we can easily create the new PV by adding the following to the editor (**remember to update with your bastion's IP**!!)

~~~bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv3
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /mnt/nfs/three
    server: 192.168.47.16 <------ CHANGEME
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
~~~

<img src="img/ui/ui-14.png"/>

And then select the "**Create**" option.

You now have an unbound volume ready for a PVC utilising CDI. 

Choose "**Storage**" and then "**Persistent Volume Claims**" and select "**Create Persistent Volume Claim**" from the upper right:

<img src="img/ui/ui-15.png"/>

Let's go ahead and enter the details for the PVC directly as YAML. Choose the "**Edit YAML**" option at the top and place the details into the editor:

~~~bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos7-ui-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
~~~

<img src="img/ui/ui-16.png"/>

Click "**Create**" to create the PVC and import the OS image.

Quickly select "**Workloads**" -> "**Pods**" and you should see the `importer-centos7-ui-nfs` pod. 

<img src="img/ui/ui-17.png"/>

Select it, and from the pods Details page that appears quickly choose the "**Logs**" tab at the top:

<img src="img/ui/ui-18.png"/>

If you were quick enough you'll see the pod import log exactly as you saw it from the CLI.

## Launching the VM

With all the pieces now in place we can now create our VM with the UI. 

Go to "**Workloads**" and choose "**Virtualization**". You'll see your running VMs (if you don't check you are in the right namespace using the namespace selector at the top of the screen) and a big blue "**Create VIrtual Machine**" option. Select the downward-facing arrow on that button and choose "**New with Wizard**"

<img src="img/ui/ui-19.png"/>

This brings up a an easy to follow wizard to launch the VM:

<img src="img/ui/ui-20.png"/>

Fill out the **General** page with the following details:

* **Name**: Choose an obvious name, here we just went with "centos7-ui-vm"
* **Description**: Something to help you remember this moment
* **Template**: We don't need to select a template for this VM
* **Source**: Choose *Disk* as we are going to select our prepared and imported PVC (we do this in a later screen)
* **Operating System**: "Red Hat Enterprise Linux 7.0 or higher" is fine for CentOS 7 :)
* **Flavor**: Choose "small" to match our other VMs.
* **Workload Profile**: Choose "server" to match our other VMs.

<img src="img/ui/ui-21.png"/>

Click **Next*.

Fill out the **Netorking** page.

Let's use our bridge adapter to access the external network again. Choose the three vertical dots next to the nic-0 (ie first) adapater and select **Edit**

<img src="img/ui/ui-22.png"/>

* Leave the **Name** and **Model** as they are.
* Set **Network** to "tuning-bridge-fixed" to use our network policy and access the external network.
* Ensure **Type** is "Bridge"
* Set **MAC Address** to `de:ad:be:ef:00:02` This ensures that things work within RHPDS

<img src="img/ui/ui-23.png"/>

And Click "**Save**" and then "**Next**"

You are now on the **Disks** setup.

Choose "**Add Disk** and fill out as follows:

* Set **Source** to "Attach Disk" so we can use an already created diks (our PVC!)
* Set **Persistent Volume Claim** to the "centos7-ui-nfs" PVC we created and loaded with Centos7!
* Leave **Name** and **Interface** as the defaults

<img src="img/ui/ui-24.png"/>

Choose **Add**.

Before clicking Next ensure you set the "**Boot Source**" to "disk-0" so we use the PVC to boot.

<img src="img/ui/ui-25.png"/>

Choose **Next** to move to the "**Advanced**" section.

Here we will set our cloud-initi config to run. Choose the "**Custom Script" radio button and enter the cloud-init details into the box:

~~~bash
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
~~~

<img src="img/ui/ui-26.png"/>

Next select "**Next**" to move to the next screen.

There is no "**Virtual Hardware**" to add, so simplu click "**Next**" again at this screen.

You can now "Review and confirm your settings." 

<img src="img/ui/ui-27.png"/>

For the moment, leave the "**Start virtual machine on creation**" **UNTICKED**. This prevents the VMI from starting.

<img src="img/ui/ui-28.png"/>

Select "See virtual machine details" to review the machine's details.

As a side note, jump back to the terminal and run `oc get vm` and `oc get vmi` right now:

~~~bash
[~] $ oc get vm
NAME                 AGE    VOLUME
centos7-masq         142m
centos7-ui-vm        72s
centos8-server-nfs   3h4m
[~] $ oc get vmi
NAME                 AGE    PHASE     IP                 NODENAME
centos7-masq         142m   Running   10.131.0.16        cluster-august-mssw8-worker-6cpjw
centos8-server-nfs   3h4m   Running   192.168.47.16/24   cluster-august-mssw8-worker-6cpjw
~~~

Feel free to have a look around before starting the VM.

When ready, and back on the console and at the new VM's "Details" page select "**Actions**" -> "**Start Virtual Machine**"

<img src="img/ui/ui-29.png"/>

The VM will start!

You can watch this process from a number of areas, check the "Console" and "Events" among other things!

<img src="img/ui/ui-30.png"/>

<img src="img/ui/ui-31.png"/>

Eventuall the VMI is shown as running on the "Details" page:

<img src="img/ui/ui-32.png"/>

Give it a few minutes more and the node's "public" IP address will report as well:

<img src="img/ui/ui-33.png"/>

You can now use that to connect to the running NGINX via your bastion-connected browser or SSH to the machine from the lab's terminal! (login with centos/redhat as per cloud-init!)

<img src="img/ui/ui-34.png"/>

Console access is available via `virtctl` or from the UI directly:

<img src="img/ui/ui-35.png"/>

### Want to try more?

If you'd like to try some more things there is one more NFS share available as well as the hostpath provisioner. 

Why not try some/all/none of the following:

* Create a PV for the /mnt/nfs/four share on your bastion and use it.

* Utilize the URL "Source" option of the VM wizard to import an image (Centos? RHEL? Windows? Your own?)

 <img src="img/ui/ui-36.png"/>
 
* Create a new VM and use OpenShift's masquerading network option.

 <img src="img/ui/ui-37.png"/>
 
* Create services via the UI!

 <img src="img/ui/ui-38.png"/>
 
Create Routes via the UI!

 <img src="img/ui/ui-39.png"/>
 
**Your environment has resources to run a few more VMs so have a play and see what you can learn!**

Check out the [documentation](https://docs.openshift.com/container-platform/4.5/virt/about-virt.html) for more use case ideas!

## The End

Thanks for participating in our lab. We hope you found it informative and educational!

As mentioned, if you have found errors or omissions we want to know about them and fix them! Submit your pull requests to the RHPDS [branch](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/rhpds) or email (field-engagement@redhat.com) the team directly so we can work together!

Any and all feedback is warmly welcomed!

Happy virtualizing!