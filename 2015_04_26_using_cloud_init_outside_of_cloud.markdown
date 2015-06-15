# Using Cloud-Init Outside of Cloud {#cloud_init}

In EC2 and OpenStack cloud environments *user data* can be passed to the cloud instance to customize the cloud instance on the first boot. But what if your virtual machine doesn't run in the cloud environment? In this article we're going to configure our virtual machines with user data regardless if they're running in the cloud or not.

<!-- more -->

# Introducing Cloud-Init {#cloud_init_intro}

[Cloud-Init](https://cloudinit.readthedocs.org/en/latest/ "Cloud-Init documentation") is a tool that handles early initialization of a cloud instance. The `cloud-init` RPM package should be installed on the disk image which the cloud instance is going to boot up from. The package installs init scripts into `/etc/rc.d/init.d` that makes Cloud-Init run early during the system initialization. Cloud-Init obtains user data passed to it by the cloud software and executes them. User data contains a set of configuration tasks for the cloud instance. For example, Cloud-Init can update machine's hostname, configure `/etc/hosts`, create users, configure SSH authorized keys, resize filesystems, manage disk mounts, run user-defined scripts and [much more](https://cloudinit.readthedocs.org/en/latest/topics/examples.html "Cloud config examples").

> Even if you're not running your virtual machines in the cloud environment it's worth it to deploy Clout-Init.

Every cloud software comes with its own mechanism of how to pass the user data to the cloud instance. For example, EC2 provides a *magic IP* from which the instance can download its user data. OpenStack cloud attaches a special *config drive* to the cloud instance containing the user data to be consumed by Clout-Init. In order to pass the user data to our virtual machine let's go the OpenStack way and assemble a minimum config drive.

# Config drive assembly {#cloud_init_drive}

First, we're going to prepare the following file structure for our config drive:

~~~
config_drive
config_drive/openstack
config_drive/openstack/latest
config_drive/openstack/2012-08-10
config_drive/openstack/2012-08-10/meta_data.json
config_drive/openstack/2012-08-10/user_data
~~~

Start by creating directories and the `latest` symbolic link like this:

~~~{.sh}
mkdir config_drive
mkdir -p config_drive/openstack/2012-08-10
ln -s 2012-08-10 config_drive/openstack/latest
~~~

Next create a minimum metadata file required by Cloud-Init. I'm using a fully qualified domain name of the virtual machine as its UUID:

~~~{.sh}
cat > config_drive/openstack/latest/meta_data.json << EOF
{
    "uuid": "myinstance.mydomain.com"
}
EOF
~~~

Cloud-Init supports many [formats](https://cloudinit.readthedocs.org/en/latest/topics/format.html "Cloud-Init user data formats") for scripts within user data. One of the most popular formats is the *cloud-config* file format. Let's create a cloud-config script that adds our SSH public key to the authorized keys for the user `root` on the virtual machine. We can then login into the virtual machine as user root without using a password. If you don't have a public-private SSH key pair you can quickly generate it using `ssh-keygen` command:

~~~{.sh}
ssh-keygen -f mykey
~~~

Now create a `user_data` file with the configuration instructions for Cloud-Init. In the following code block replace the value of the `ssh-authorized-keys` field with the content of your generated `mykey.pub` file:

~~~{.sh}
cat > config_drive/openstack/latest/user_data << EOF
#cloud-config
fqdn: myinstance.mydomain.com
users:
  - name: root
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDNH8Qwn4raGR1f9fvjbZe/GXM2N9Mh+eWlsFoYpcU4H5qf5YxT5CUo7BaTOgeE5geHyzxJQmCQlvoxcW3qkcjBJvVgEsTrrnX7KYS8BszvT4AMIuG2Za8f7myubXd6zYfj74XYhutUsPz7x2TEp9ZqbVkWcaElrQFxF2AzF7dV1RGntpPKyISqem70En8RYpGY514OLZ9TQDBYjbw8tfPuDd9mznXnWOZ34fPtP7+QDvOMFuA4tXsBpHj99/cbC0ViwzZtvb1QtY7dv9OFDgCRadw81+SKtzXctQ2rCYkb0huc0BCE7kLzinzlO62Znd+N1d+tpLAwP6i8Z5ZMXIJj user@machine
EOF
~~~

The file structure for our config drive is ready. Let's generate an ext2 filesystem and copy the files to it. The `virt-make-fs` utility from the `libguestfs-tools` package can help us with that:

~~~{.sh}
virt-make-fs config_drive disk.config
~~~

In order for Cloud-Init to detect the attached drive as config drive the filesystem on the config drive needs to be labeled `config-2`. You can use `e2label` command from the `e2fsprogs` package to label your config drive:
~~~{.sh}
e2label disk.config config-2
~~~

# Cloud-Init in action {#cloud_init_action}

On my Linux host I'm running [libvirt](http://libvirt.org/ "Libvirt - The virtualization API") to ease the management of virtual machines. You can install it by running `sudo yum install libvirt`. There is a handy command-line utility `virsh` which comes  with libvirt in the extra package `libvirt-client`.

Let's create a virtual machine with the config drive attached. As a virtual machine boot image I'm using a CentOS-6 image from [cloud.centos.org](http://cloud.centos.org/centos/6/images/ "CentOS-6 cloud images") which comes with Cloud-Init built in. Make sure that your virtual machine boot image has Cloud-Init installed. Following is a virtual machine definition file for the CentoOS-6 virtual machine. You might need to change the location of the disk image files and save it as `CentOS-6.xml`:

~~~{.xml}
<domain type='kvm'>
  <name>CentOS-6</name>
  <memory unit='KiB'>2097152</memory>
  <os>
    <type>hvm</type>
  </os>
  <devices>
    <disk type='file' device='disk'>
      <driver name="qemu" type="qcow2"/>
      <source file='/tmp/CentOS-6-x86_64-GenericCloud.qcow2'/>
      <target bus="virtio" dev="vda"/>
    </disk>
    <disk type='file' device='disk'>
      <driver name="qemu" type="raw"/>
      <source file='/tmp/disk.config'/>
      <target bus="virtio" dev="vdb"/>
    </disk>
    <interface type='network'>
      <source network='default'/>
    </interface>
    <serial type="file">
      <source path="/tmp/CentOS-6.log"/>
    </serial>
  </devices>
</domain>
~~~

Okay, everything is ready, let's launch our Cloud-Init enabled CentOS-6 virtual machine:
~~~{.sh}
sudo virsh define CentOS-6.xml
sudo virsh start CentOS-6
~~~
If everything went fine you can watch the console output of the booting virtual machine at `/tmp/CentOS-6.log`. Cloud-Init will print out the IP address obtained by the virtual machine (192.168.122.165 in my case) where we can login as root using the generated private key:
~~~{.sh}
ssh -i testkey root@192.168.122.165
~~~
Congratulations, your virtual machine has just been configured by Cloud-Init the same way as any other virtual machine running in the cloud environment.
