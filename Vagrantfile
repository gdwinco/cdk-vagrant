# -*- mode: ruby -*-
# vi: set ft=ruby :

# The private network IP of the VM. You will use this IP to connect to OpenShift.
# This variable is ignored for Hyper-V provider.
PUBLIC_ADDRESS="10.1.2.2"

OCP_USER = ENV['OCP_USER'] || 'user'
OCP_PASSWORD = ENV['OCP_PASSWORD'] || 'r3dh4t'

# Number of virtualized CPUs
VM_CPU = ENV['VM_CPU'] || 2

# Amount of available RAM
VM_MEMORY = ENV['VM_MEMORY'] || 6144

# Validate required plugins
REQUIRED_PLUGINS = %w(vagrant-service-manager vagrant-registration vagrant-sshfs)
errors = []

def message(name)
  "#{name} plugin is not installed, run `vagrant plugin install #{name}` to install it."
end
# Validate and collect error message if plugin is not installed
REQUIRED_PLUGINS.each { |plugin| errors << message(plugin) unless Vagrant.has_plugin?(plugin) }
unless errors.empty?
  msg = errors.size > 1 ? "Errors: \n* #{errors.join("\n* ")}" : "Error: #{errors.first}"
  fail Vagrant::Errors::VagrantError.new, msg
end

Vagrant.configure(2) do |config|
  config.vm.box = 'cdk'

  config.vm.provider "virtualbox" do |v, override|
    v.memory = VM_MEMORY
    v.cpus   = VM_CPU
    v.customize ["modifyvm", :id, "--ioapic", "on"]
    v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  config.vm.provider "libvirt" do |v, override|
    v.memory = VM_MEMORY
    v.cpus   = VM_CPU
    v.driver = "kvm"
    v.suspend_mode = "managedsave"
  end

  config.vm.network "private_network", ip: "#{PUBLIC_ADDRESS}"

  # vagrant-registration
  if ENV.has_key?('SUB_USERNAME') && ENV.has_key?('SUB_PASSWORD')
    config.registration.username = ENV['SUB_USERNAME']
    config.registration.password = ENV['SUB_PASSWORD']
  end

  # Proxy Information from environment
  ENV['PROXY'] ? (config.registration.proxy = PROXY = ENV['PROXY']) : PROXY = ''
  ENV['PROXY_USER'] ? (config.registration.proxyUser = PROXY_USER = ENV['PROXY_USER']) : PROXY_USER = ''
  ENV['PROXY_PASSWORD'] ? (config.registration.proxyPassword = PROXY_PASSWORD = ENV['PROXY_PASSWORD']) : PROXY_PASSWORD = ''

  # vagrant-sshfs
  config.vm.synced_folder '.', '/vagrant', disabled: true
  if Vagrant::Util::Platform.windows?
    target_path = ENV['USERPROFILE'].gsub(/\\/,'/').gsub(/[[:alpha:]]{1}:/){|s|'/' + s.downcase.sub(':', '')}
    config.vm.synced_folder ENV['USERPROFILE'], target_path, type: 'sshfs', sshfs_opts_append: '-o umask=000 -o uid=1000 -o gid=1000'
  else
    config.vm.synced_folder ENV['HOME'], ENV['HOME'], type: 'sshfs', sshfs_opts_append: '-o umask=000 -o uid=1000 -o gid=1000'
  end
  config.vm.provision "shell", inline: <<-SHELL
    sudo setsebool -P virt_sandbox_use_fusefs 1
  SHELL

  # prevent the automatic start of openshift via service-manager by just enabling Docker
  config.servicemanager.services = "docker"

  # explicitly enable and start OpenShift
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    PROXY=#{PROXY} PROXY_USER=#{PROXY_USER} PROXY_PASSWORD=#{PROXY_PASSWORD} /usr/bin/sccli openshift
  SHELL
  
  # Fix date/time issues with VM
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    export TZ='Europe/London'
    echo "export TZ='Europe/London'" >> ~/.bash_profile
    yum install -y ntp
    systemctl enable ntpd
    systemctl start ntpd
  SHELL
  
  # Update OpenShift config to enable build pipelines
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Configuring Build Pipelines (Tech Preview)"
    echo
    sudo su -
    echo "window.OPENSHIFT_CONSTANTS.ENABLE_TECH_PREVIEW_FEATURE.pipelines = true;" >> /var/lib/openshift/openshift.local.config/master/extensions.js
    sed -i "s/autoProvisionEnabled\: false/autoProvisionEnabled\: true/g" /var/lib/openshift/openshift.local.config/master/master-config.yaml || true
    sed -i "s/extensionScripts\: null/extensionScripts\:\\n  - extensions.js/g" /var/lib/openshift/openshift.local.config/master/master-config.yaml || true
  SHELL
  
  # Create authentication / authorisation policy 
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Creating user and assigning cluster-admin role"
    echo
    sudo su -
    htpasswd -b /var/lib/openshift/openshift.local.config/master/user.htpasswd '#{ENV['OCP_USER']}' '#{ENV['OCP_PASSWORD']}'
    mkdir ~/.kube
    cp /var/lib/openshift/openshift.local.config/master/admin.kubeconfig ~/.kube/config
    oc login -u system:admin -n openshift
    oc adm policy add-cluster-role-to-user cluster-admin #{ENV['OCP_USER']}
    echo "Patching anyuid scc"
    oc patch scc/anyuid --patch '{"groups":["system:cluster-admins"]}'
  SHELL

  config.vm.provision "shell", run: "always", inline: <<-SHELL
    systemctl enable openshift 2>&1
    systemctl restart openshift
  SHELL
 
  # Clone openshift-ansible project and import Image Streams and Templates
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Importing Image Streams and Templates"
    echo
    sudo su -
    git clone https://github.com/openshift/openshift-ansible
    oc create -f openshift-ansible/roles/openshift_examples/files/examples/v1.3/db-templates -n openshift || true
    oc create -f openshift-ansible/roles/openshift_examples/files/examples/v1.3/image-streams/image-streams-rhel7.json -n openshift || true
    oc create -f openshift-ansible/roles/openshift_examples/files/examples/v1.3/quickstart-templates -n openshift || true
    oc create -f openshift-ansible/roles/openshift_examples/files/examples/v1.3/xpaas-streams -n openshift || true
    oc create -f openshift-ansible/roles/openshift_examples/files/examples/v1.3/xpaas-templates -n openshift || true
    oc create -f openshift-ansible/roles/openshift_hosted_templates/files/v1.3/enterprise -n openshift || true 
  SHELL
  
  # Pull images for CICD namespace
  # What we should be pulling:
  #     docker pull openshift3/jenkins-1-rhel7:latest
  #     docker pull openshift3/jenkins-slave-maven-rhel7:latest
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Pulling images for CICD Project"
    echo
    docker pull docker.io/openshift/jenkins-1-centos7:latest
    docker pull docker.io/openshift/jenkins-slave-maven-centos7:latest
    docker pull docker.bintray.io/jfrog/artifactory-oss:latest
  SHELL
  
  # Provision CICD namespace
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Creating CI/CD Project"
    echo
    oc login -u #{ENV['OCP_USER']} -p #{ENV['OCP_PASSWORD']}
    oc new-project cicd
    oc import-image jenkins-centos --from=docker.io/openshift/jenkins-1-centos7 --insecure=true --confirm
    oc import-image jenkins-centos-slave --from=docker.io/openshift/jenkins-slave-maven-centos7 --insecure=true --confirm
    oc import-image artifactory-oss --from=docker.bintray.io/jfrog/artifactory-oss:latest --insecure=true --confirm
  SHELL
  
  # Provision CICD Tooling
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    echo
    echo "Provision CI/CD Tooling"
    echo
    oc login -u system:admin
    oc project cicd
    oc new-app --template=jenkins-persistent -p NAMESPACE=cicd -p JENKINS_IMAGE_STREAM_TAG=jenkins-centos:latest
    oc create sa artifactory
    oc project cicd
    oc adm policy add-scc-to-user anyuid -z artifactory
    oc new-app cicd/artifactory-oss
    echo
    echo "Time for a power nap..."
    echo
    sleep 5
    echo
    echo "I'm awake!"
    echo
    oc patch dc/artifactory-oss --patch '{"spec":{"template":{"spec":{"serviceAccountName": "artifactory"}}}}'
    oc expose service artifactory-oss
  SHELL
  
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    #Get the routable IP address of OpenShift
    OS_IP=`/opt/adb/openshift/get_ip_address`
    echo
    echo "Successfully started and provisioned VM with #{VM_CPU} cores and #{VM_MEMORY} MB of memory."
    echo "To modify the number of cores and/or available memory set the environment variables"
    echo "VM_CPU and/or VM_MEMORY respectively."
    echo
    echo "You can now access the OpenShift console on: https://${OSIP}:8443/console"
    echo
    echo "To download and install OC binary, run:"
    echo "vagrant service-manager install-cli openshift"
    echo
    echo "To use OpenShift CLI, run:"
    echo "$ oc login ${OS_IP}:8443"
    echo
    echo "Configured users are (<username>/<password>):"
    echo "openshift-dev/devel"
    echo "admin/admin"
    echo #{ENV['OCP_USER']}/#{ENV['OCP_PASSWORD']}
    echo
    echo "If you have the oc client library on your host, you can also login from your host."
    echo
  SHELL
end
