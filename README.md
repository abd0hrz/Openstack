# First of I mainly followed the techandtrains.com blog to figure most of this out, thank you Gregory.
# If you are using vagrant, configure network in brdige mode so that you are not NATed and have a unique IP
# YOu can ignore the vagrant commands if you are directly setting up CentOS7 on VirtualBox.
# I recommend osboxes.org to get a prebuilt vanilla image, you are guys are very cool!.

# Install vagrant
yum install -y vagrant

# Download the box and add in vagrantboxes folder
vagrant box add CenOS7x64 https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box

# Initialize vagrant and create a vagrantconfig file, where every attribute of the box can be configured
vagrant init

# Load the instance into virtualbox
vagrant up

# After the VM has been setup and follow along
# Make sure the system is upto date  
yum update -y

# Reboot the system
shutdown -r now

# Download openstack kilo rpm
yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-kilo/rdo-release-kilo-1.noarch.rpm

# Install epel release and packstack and a few other packages
yum install -y epel-release openstack-packstack wget screen git 

# Install packages recommended by openstack dev
yum install -y python-devel openssl-devel 
yum install -y python-pip git gcc libxslt-devel mysql-devel
yum install -y postgresql-devel libffi-devel libvirt-devel graphviz yum install -y sqlite-devel
sudo pip-python install tox

# If postgres is preferred over mariadb or mysql, install postgres
# client and server
yum install -y postgresql-server 
yum install -y postgresql-client

# Start the ssh daemon
sudo systemctl start sshd

# Stop and Remove default network manager as neutron will manage network
service NetworkManager stop
yum remove NetworkManager

# Network would be turned off, so start network and enable on reboot
systemctl start network.service
systemctl enable network.service

# Check status 
chkconfig network on
systemctl status network.service

# Change the /etc/hosts file to make sure that hostname and associated ip is present.
# This part is not mentioned in the RDO documentation.
# I lost a lot of time here, it is a trial change, but without this rabbitmqserver will not start up 
# when packstack is trying to turn on all the services

echo "127.0.0.1 localhost "hostname" >> /etc/hosts
#example:: echo "127.0.0.1 localhost osboxes" >> /etc/hosts

#Turn on rabbitmq-server and enable on reboot
systemctl start rabbitmq-server
systemctl enable rabbitmq-server

# Now its time to start packstack, install as a normal user (will be prompted for password.
# if not want to be disturned, just create a env variable for the sudo password
packstack --install-hosts=127.0.0.1 --use-epel=n --provision-demo=n 

# Now openstack should be setup and running on 127.0.0.1
# login credentials would be stored in  keystonerc_admin
# Have fun!!, or not ..

# Download, extract and load a CoreOS image 
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
bunzip2 coreos_production_openstack_image.img.bz2
glance image-create --name "CoreOS" --container-format bare \
--disk-format qcow2 --file coreos_production_openstack_image.img \
--is-public True

# Setup virtual proxy using VNC
openstack-config --set /etc/nova/nova.conf DEFAULT \ novncproxy_base_url http://192.168.1.24:6080/vnc_auto.html

# Using QEMU instead of KVM as the hypervisor as  we are running on a VM
openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu
openstack-config --set /etc/nova/nova.conf DEFAULT compute_driver \ libvirt.LibvirtDriver setsebool -P virt_use_execmem on \
ln -s /usr/libexec/qemu-kvm /usr/bin/qemu-system-x86_64
service libvirtd restart
service openstack-nova-compute restart

# Finally all done!!
# Now 
# To start using coreOS need to setup discovery tokens,
# configure etcd etc which can be done through the coreos official documentation
# Now have fun!!
