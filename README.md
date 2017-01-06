# cdk-vagrant
A modified Vagrantfile for the OpenShift CDK. Assumes you have the latest Container Development Kit image from https://developers.redhat.com/

It uses the CentOS7 images for Jenkins / Slaves because they use more up-to-date plugins than the shipped RHEL7 image (namely, fixes to address https://github.com/fabric8io/openshift-jenkins-sync-plugin/issues/85) - This was easier than getting Jenkins to update shedloads of plugins at runtime. When the update happens, I'll update the Vagrantfile.

# Useage

Define the following environment variables to configure the image:

* OCP_USER - the user we create to carry out the operations. Defaults to 'user'
* OCP_PASSWORD - the password for the user. Defaults to 'r3dh4t' (obviously).
* OCP_CICD_PROJECT - the name of the cicd project we want to create. Defaults to 'cicd'.
* TZ - the time zone we're operating in. Defaults to 'Europe/London'.

# What it does
Assuming you have the latest CDK image from developers.redhat.com, this Vagrantfile will:

* Fix the fact that the CDK was convinced you lived on the US East Coast, even if you weren't. Defaults to 'Europe/London' but is configurable (and should take the value of 'TZ' if defined).
* Remove the anyuid scc from system:authenticated users so the security model behaves more like a full-fat cluster
* Enable Tech Preview Build Pipelines out of the box
* Create a new cluster user
* Import all the missing xPaaS image streams and templates
* Create a CICD project, and spin up persistent Jenkins and Artifactory instances.

**NOTE: It sometimes takes Artifactory a few minutes to resolve its image from the ImageStream. Please be patient.**
