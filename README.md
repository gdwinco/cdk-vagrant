# cdk-vagrant
A modified Vagrantfile for the OpenShift CDK. It uses the CentOS7 images for Jenkins/Slaves  because they use more up-to-date plugins than the shipped RHEL7 image - This was easier than getting Jenkins to update shedloads of plugins at runtime. When the update happens, I'll update the Vagrantfile.

# Useage

Define the following environment variables to configure the image:

* OCP_USER - the user we create to carry out the operations. Defaults to 'user'
* OCP_PASSWORD - the password for the user. Defaults to 'r3dh4t' (obviously).

# What it does
Assuming you have the latest CDK image from developers.redhat.com, this Vagrantfile will:

* Fix the fact that the CDK was convinced you lived on the US East Coast
* Remove the anyuid scc from system:authenticated users so the security model behaves more like a full-fat cluster
* Enable Tech Preview Build Pipelines out of the box
* Create a new cluster user
* Import all the missing xPaaS image streams and templates
* Create a CICD project, and spin up a Jenkins and an Artifactory instance *(TODO - Provision storage automatically for this)*
