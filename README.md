## OpenShift Virtualization Hands-on Lab (RHPDS)

**Authors**: [Rhys Oxenham](mailto:roxenham@redhat.com) and [August Simonelli](mailto:asimonel@redhat.com)

**PLEASE NOTE THIS LAB IS IS NOT YET AVAILABLE FOR USE IN RHPDS.**

# Welcome!

Welcome to our hands-on OpenShift Virtualization lab for RHPDS. 

You first need to deploy your lab through RHPDS. Once you do that you will receive full access details and instructions for your specific environment.

To launch the lab in RHPDS ... TBA.

> **NOTE**: We also have a branch that can be used on your own hardware which includes deployment scripts for a completely self-contained training. This is available from the [main branch of this lab's repo](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/master).

# Lab environment

The lab includes a self-hosted OpenShift Virtualization environment and a hands-on self-paced lab guide based on [OpenShift homeroom](https://github.com/openshift-homeroom).

> **NOTE**: For the purposes of this repo and the labs themselves, any reference to "CNV", "Container-native Virtualization" and "OpenShift Virtualization", and "KubeVirt" can be used interchangeably.

# Lab content

The lab uses official Red Hat downstream components, where **OpenShift Virtualization** is now the official feature name of the packaged up [Kubevirt project](https://kubevirt.io/) within the OpenShift product. 

The lab runs you through the following OpenShift Virtualization tasks:

* **[Validating the OpenShift deployment](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/validation.md)**
* **[Deploying OpenShift Virtualization](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/deploy-cnv.md)**
* **[Setting up Storage for OpenShift Virtualization](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/storage-setup.md)**
* **[Setting up Networking for OpenShift Virtualization](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/network-setup.md)**
* **[Deploying Test Workloads](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/deploy-workloads.md)**
* **[Cloning Workloads](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/cloning.md)**
* **[Performing Live Migrations and Node Maintenance](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/live-migration.md)**
* **[Utilising pod networking for VM's](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/masquerade.md)**
* **[Using the OpenShift Web Console with OpenShift Virtualization](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/docs/workshop/content/console.md)** (beta, under review!)

As mentioned above, the entire setup is provided contained within infrastructure provided by RHPDS. This means you can easily deploy the lab, follow some simple setuo instructions, and you will have your own OpenShift cluster to work on with full admin access. 

The deployment is visualised as follows:

<center>
    <img src="docs/workshop/content/img/labarch.png"/>
</center>

Within this environment you can access all aspects of the lab through the deployed lab guide. You receive access to that guide with the RHPDS welcome email.

### Getting Started

Your RHPDS email will provide full intructions for setting up your access.

### Contributing

**We very much welcome contributions and pull requests!**
