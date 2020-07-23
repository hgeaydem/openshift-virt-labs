On the right hand side where the web terminal is, let's see if we can check the nodes:

~~~bash
[~] $ oc get nodes
NAME                                STATUS     ROLES    AGE    VERSION
cluster-august-44qqh-master-0       Ready      master   3h6m   v1.18.3+3107688
cluster-august-44qqh-master-1       Ready      master   3h6m   v1.18.3+3107688
cluster-august-44qqh-master-2       Ready      master   3h6m   v1.18.3+3107688
cluster-august-44qqh-worker-kztfc   NotReady   worker   172m   v1.18.3+3107688
cluster-august-44qqh-worker-tvtl6   Ready      worker   172m   v1.18.3+3107688
~~~

> **NOTE**: You will see different naming than those above with your GUID visible in all hostnames. What is most important is you see 3 masters and 2 workers.

Next let's validate the version that we've got deployed, and the status of the cluster operators:

~~~bash
[asimonel-redhat.com@bastion cnv]$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.5.2     True        False         17h     Cluster version is 4.5.2
~~~

This cluster is a 4.5.2 deployment and is currently stable (it not showing as "progessing" still). Let's next review the cluster operators and their status. We should expect them to all be available and also not "progressing" or "degraded."

~~~bash
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.2     True        False         False      17h
cloud-credential                           4.5.2     True        False         False      17h
cluster-autoscaler                         4.5.2     True        False         False      17h
config-operator                            4.5.2     True        False         False      17h
console                                    4.5.2     True        False         False      17h
csi-snapshot-controller                    4.5.2     True        False         False      17h
dns                                        4.5.2     True        False         False      17h
etcd                                       4.5.2     True        False         False      17h
image-registry                             4.5.2     True        False         False      17h
ingress                                    4.5.2     True        False         False      17h
insights                                   4.5.2     True        False         False      17h
kube-apiserver                             4.5.2     True        False         False      17h
kube-controller-manager                    4.5.2     True        False         False      17h
kube-scheduler                             4.5.2     True        False         False      17h
kube-storage-version-migrator              4.5.2     True        False         False      17h
machine-api                                4.5.2     True        False         False      17h
machine-approver                           4.5.2     True        False         False      17h
machine-config                             4.5.2     True        False         False      17h
marketplace                                4.5.2     True        False         False      17h
monitoring                                 4.5.2     True        False         False      17h
network                                    4.5.2     True        False         False      17h
node-tuning                                4.5.2     True        False         False      17h
openshift-apiserver                        4.5.2     True        False         False      17h
openshift-controller-manager               4.5.2     True        False         False      17h
openshift-samples                          4.5.2     True        False         False      17h
operator-lifecycle-manager                 4.5.2     True        False         False      17h
operator-lifecycle-manager-catalog         4.5.2     True        False         False      17h
operator-lifecycle-manager-packageserver   4.5.2     True        False         False      17h
service-ca                                 4.5.2     True        False         False      17h
storage                                    4.5.2     True        False         False      17h
~~~


### Making sure OpenShift works

OK, so this is likely something that you've all done before, and it's hardly very exciting, but let's have a little bit of fun. Let's deploy a nifty little application inside of a pod and use it to verify that the OpenShift cluster is functioning properly; this will involve building an application from source and exposing it to your web-browser. We'll use the **s2i** (source to image) container type:

~~~bash
$ oc project default
Now using project "default" on server "https://api.cnv.example.com:6443".

$ oc new-app \
	nodeshift/centos7-s2i-nodejs:12.x~https://github.com/vrutkovs/DuckHunt-JS

(...)

--> Creating resources ...
    imagestream.image.openshift.io "centos7-s2i-nodejs" created
    imagestream.image.openshift.io "duckhunt-js" created
    buildconfig.build.openshift.io "duckhunt-js" created
    deploymentconfig.apps.openshift.io "duckhunt-js" created
    service "duckhunt-js" created
--> Success
    Build scheduled, use 'oc logs -f bc/duckhunt-js' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/duckhunt-js'
    Run 'oc status' to view your app.
~~~

Our application will now build from source, you can watch it happen with:

~~~bash
$ oc logs duckhunt-js-1-build -f
(...)

Successfully pushed image-registry.openshift-image-registry.svc:5000/default/duckhunt-js:latest@sha256:4d0186040826a4be9d678459c5d6831e107a60c403d65a0da77fb076ff89084c
Push successful
~~~

> **NOTE**: You may get an error saying "Error from server (BadRequest): container "sti-build" in pod "duckhunt-js-1-build" is waiting to start: PodInitializing"; you were just too quick to ask for the log output of the pods, simply re-run the command.

You can check if the Duckhunt pod has finished building and is `Running`, if it's still showing as `ContainerCreating` just give it a few more seconds:

~~~bash
$ oc get pods
NAME                   READY   STATUS      RESTARTS   AGE
duckhunt-js-1-build    0/1     Completed   0          5m17s
duckhunt-js-2-deploy   0/1     Completed   0          3m8s
duckhunt-js-2-sbcgr    1/1     Running     0          2m6s     <-- this is the one!
~~~

Now expose the application (via the service) so we can route to it from the outside...

~~~bash
$ oc expose svc/duckhunt-js
route.route.openshift.io/duckhunt-js exposed

$ oc get route duckhunt-js
NAME          HOST/PORT                                  PATH   SERVICES      PORT       TERMINATION   WILDCARD
duckhunt-js   duckhunt-js-default.apps.cnv.example.com          duckhunt-js   8080-tcp                 None
~~~

You should be able to open up the application in the same browser that you're reading this guide from, either copy and paste the address, or click this clink: [http://duckhunt-js-default.apps.cnv.example.com](http://duckhunt-js-default.apps.cnv.example.com). If your OpenShift cluster is working as expected and the application build was successful, you should now be able to have a quick play with this... good luck ;-)

<img src="img/duckhunt.png"/>

Now, if you can tear yourself away from the game, let's actually start working with OpenShift Virtualization, first let's just clean up the default project ...

~~~bash
$ $ oc delete deployment/duckhunt-js bc/duckhunt-js svc/duckhunt-js route/duckhunt-js
deployment.apps "duckhunt-js" deleted
buildconfig.build.openshift.io "duckhunt-js" deleted
service "duckhunt-js" deleted
route.route.openshift.io "duckhunt-js" deleted
~~~