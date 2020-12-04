# OpenShift Virtualization Hands-on Lab (RHPDS)

**Authors**: [Rhys Oxenham](mailto:roxenham@redhat.com) and [August Simonelli](mailto:asimonel@redhat.com)

## Welcome!

Welcome to our hands-on OpenShift Virtualization lab for the Red Hat Product Demo System (RHPDS) self-service environment. 

> **NOTE**: We also have a branch that can be used on your own hardware which includes deployment scripts for a completely self-contained training. This is available from the [main branch of this lab's repo](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/master).

## Lab environment

The lab includes a self-hosted OpenShift Virtualization environment and a hands-on self-paced lab guide utilizing [OpenShift homeroom](https://github.com/openshift-homeroom).

The lab content is presented within your browser in three easy to use sections consisting of the following sections: navigation, lab steps, and working environment. 

Through this browser-based environment you'll have access to an OpenShift CLI environment *as well as* the OpenShift UI (console).

All labs steps are run from *within* this fully-contained browser based environment.

## Ordering your lab environment
Labs are ordered from within the [Red Hat Product Demo System (RHPDS)](https://rhpds.redhat.com/catalog/explorer) environment. Once you login to RHPDS go to **Service Catalogs > OpenShift Demos** and choose **OpenShift Virtualization (CNV) Lab**:

<center>
    <img src="docs/workshop/content/img/intro-rhpds-catalog.png"/>
</center>

Select **Order** to request your lab.

## Accessing your lab environment
You will receive three progress emails from RHPDS while your lab environment is prepared. After about an hour the **third email** indicates the build is done and contains acess a lot of details.

As mentioned, the lab is entirely self-contained in it's own web application. So while the email gives you full details for your OCP install, you need to find the **CNV Lab Workbook** link to access it.

IMAGE TO BE ADDED SHORTLY

<center>
    <img src="docs/workshop/content/img/intro-rhpds-email-cnvlablink.png-OFF"/>
</center>

**But keep this email!** 

You do not need this info for the majority of the lab as all the required access (both OCP CLI and OCP GUI) is available via the single **CNV Lab Workbook** URL *without authentication*. 

However, for a later lab you *will* access the bastion via SSH to utilise a proxy server on there.

**Once the email has arrived follow the link for the CNV Lab Workbook and get started there.** For your information an overview of the lab content is below but please use the self-hosted guide to do the labs.


### Lab architecture 

The entire setup is contained within infrastructure provided by RHPDS. All you need to do is Order the lab and access the CNV Workbook Link!

The deployment is visualised as follows:

<center>
    <img src="docs/workshop/content/img/labarch.png"/>
</center>

> **REMINDER**: **Within this environment you can access all aspects of the lab through the deployed lab guide which is available via the CNV Workbook Link in the RHPDS email.**

> **NOTE**: For the purposes of this repo and the labs themselves, any reference to "CNV," "Container-native Virtualization," "OpenShift Virtualization," and "KubeVirt" can be used interchangeably.

# Contributing

**We very much welcome contributions and pull requests! And if you have any questions please reach out directly to us at field-engagement@redhat.com**
