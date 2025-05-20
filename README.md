# Packstack-OpenStack Setup on CentOS (VirtualBox)

This guide outlines the steps to set up **Packstack (OpenStack Kilo)** on a **CentOS 7** virtual machine using **VirtualBox**. I faced several challenges while setting it up, so I’ve compiled this to help others get started smoothly.

> 💡 If you’re using **Vagrant**, configure the network in **bridge mode** to avoid NAT and get a unique IP address.  
> 💡 You can skip the Vagrant instructions if you're setting up CentOS 7 directly on VirtualBox.  
> 🔗 I recommend [osboxes.org](https://www.osboxes.org/) for ready-to-use CentOS images.

---

## Step-by-Step Installation

### 1. Vagrant Setup (Optional)

```bash
yum install -y vagrant

# Add CentOS 7 box
vagrant box add CenOS7x64 https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box

# Initialize Vagrant
vagrant init

# Start the VM
vagrant up
```

---

### 2. Update System and Reboot

```bash
yum update -y
shutdown -r now
```

---

### 3. Install Required Repositories and Packages

```bash
# Add OpenStack Kilo repository
yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-kilo/rdo-release-kilo-1.noarch.rpm

# Install Packstack and other utilities
yum install -y epel-release openstack-packstack wget screen git

# Recommended developer tools
yum install -y python-devel openssl-devel python-pip gcc git libxslt-devel mysql-devel \
postgresql-devel libffi-devel libvirt-devel graphviz sqlite-devel

# Python packages
pip install tox
```

---

### 4. Optional: PostgreSQL (Instead of MariaDB)

```bash
yum install -y postgresql-server postgresql-client
```

---

### 5. Network Configuration

```bash
# Start SSH
systemctl start sshd

# Disable and remove NetworkManager (neutron will handle networking)
service NetworkManager stop
yum remove -y NetworkManager

# Enable standard network service
systemctl start network.service
systemctl enable network.service
chkconfig network on
systemctl status network.service
```

---

### 6. Hostname Configuration

Edit `/etc/hosts` and add:

```bash
127.0.0.1   localhost your-hostname
# Example:
# 127.0.0.1   localhost osboxes
```

This is crucial for RabbitMQ to start properly during Packstack installation.

---

### 7. Start RabbitMQ

```bash
systemctl start rabbitmq-server
systemctl enable rabbitmq-server
```

---

### 8. Install OpenStack with Packstack

```bash
packstack --install-hosts=127.0.0.1 --use-epel=n --provision-demo=n
```

After completion, login credentials will be stored in the file `keystonerc_admin`.

---

## Optional: Load CoreOS Image into Glance

```bash
wget http://stable.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
bunzip2 coreos_production_openstack_image.img.bz2

glance image-create --name "CoreOS" --container-format bare \
  --disk-format qcow2 --file coreos_production_openstack_image.img \
  --is-public True
```

---

## Configure VNC and QEMU (Virtualized Environments)

```bash
# Set VNC proxy URL
openstack-config --set /etc/nova/nova.conf DEFAULT \
  novncproxy_base_url http://192.168.1.24:6080/vnc_auto.html

# Use QEMU instead of KVM (running inside a VM)
openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu
openstack-config --set /etc/nova/nova.conf DEFAULT compute_driver libvirt.LibvirtDriver

# Enable required SELinux setting
setsebool -P virt_use_execmem on

# Symlink QEMU binary if needed
ln -s /usr/libexec/qemu-kvm /usr/bin/qemu-system-x86_64

# Restart services
service libvirtd restart
service openstack-nova-compute restart
```

---

## Done!

Your Packstack OpenStack environment should now be up and running on **127.0.0.1**.

To continue with CoreOS or additional OpenStack services, refer to official documentation.
