
# Ovirt from scratch using Glusterfs

==**[All the steps mentioned below must be done on all the servers unless instructed otherwise]**==

## Preparing the OS

### Creating the instalation media

* Ovirt is only support on RedHat and CentOS (Not on Ubuntu). I used CentOS
* Used `CentOS-7-x86_64-Minimal-1804.iso` to create the CentOS installation
	* Lighter but without GUI
	* Did not cause issue while creating a USB bootable drive
	* Takes less time to create
* Used `dd` to create a bootable USB installer
	* Do not use `UNetbootin` or "Startup Disk creator". They can cause error
	* The name of the bootable USB drive must be proper (`CentOS-7-x86_64-installer`)
* Use `md5hash` to check the ISO

### During installation

While installing CentOS
* Did manual partition
* I didn't create the gluster partition. It was unallocated
* I applied the IP address. Make sure to switch to `manual` from `dhcp`

### Post-installation

#### Apply proxy everywhere

* Applied proxy in `/etc/yum.conf`. Add the line at the end in the mentioned file
	`proxy=http://172.16.2.30:8080`
* Applied proxy in `/etc/environment`
	```
	export http_proxy=http://172.16.2.30:8080
	export https_proxy=http://172.16.2.30:8080
	export ftp_proxy=http://172.16.2.30:8080
	export no_proxy=10.0.0.0/8,10.5.20.121,management.org,localhost,127.0.0.1,192.168.0.0/16
	export NO_PROXY=$no_proxy
	```

#### Change interface name to `eth0`

* Edit the file `/etc/default/grub`
	* Add the string `net.ifnames=0 biosdevname=0` to the line `GRUB_CMDLINE_LINUX`
* Regenate the grub file
	`grub2-mkconfig -o /boot/grub2/grub.cfg`
* The default interface name is `p8p1`. Edit the file `/etc/sysconfig/network-scripts/ifcfg-p8p1`
	* Change device name from `p8p1` to `eth0`
* Rename the above file
	`mv /etc/sysconfig/network-scripts/ifcfg-p8p1 /etc/sysconfig/network-scripts/ifcfg-eth0`
* Reboot the machine

[(Source)](https://www.thegeekdiary.com/centos-rhel-7-how-to-modify-network-interface-names/)

#### Update CentOS
 `yum update -y`

#### Installing GUI

* Installed GUI only in the management PC and the client PC
	`yum groupinstall "GNOME Desktop" -y`
* Then I had to enable the GUI
	`ln -sf /lib/systemd/system/runlevel5.target /etc/systemd/system/default.target`
* For the other servers, I didn't install the GUI. It's better to be light

[(Source^1^)](https://www.itzgeek.com/how-tos/linux/centos-how-tos/install-gnome-gui-on-centos-7-rhel-7.html) [(Source^2^)](https://unix.stackexchange.com/a/264097/209317)

#### Install some utilities and enable/disable some services
* By default, `ifconfig` is not installed. Installed it by
	`yum install net-tools -y` 
* Installed `htop` for system monitoring
	`yum install epel-release -y`
	`yum install htop -y`
* Installed `system-storage-manager` for preparing gluster partitions
	`yum install system-storage-manager -y`
* Start the firewall. Ovirt requirement [(Source)](https://bugzilla.redhat.com/show_bug.cgi?id=1540451)
	`systemctl unmask firewalld && systemctl start firewalld`
* Add rule for firewall to allow gluster traffic [(Source)](https://bugzilla.redhat.com/show_bug.cgi?id=1327590)
	`firewall-cmd --zone=public --add-service=glusterfs --permanent`
* Install a local DNS server. (_Don't know why this is required. Hosted Engine might need it_)
	`yum install dnsmasq -y`
	`systemctl enable dnsmasq && systemctl start dnsmasq`
* Add self to the DNS server list. Edit the file `/etc/resolv.conf`. (_Mandatory step_) Add to the top of the file
	`nameserver 127.0.0.1`
	You can also the give the `eth0` IP (_Should not be an issue_)
* Install gluster and ovirt (_IMPORTANT!_)
	`yum install -y ovirt-hosted-engine-setup glusterfs-server vdsm-gluster glusterfs`
* Disable the network manager [(Source)](https://www.thegeekdiary.com/centos-rhel-7-how-to-disable-networkmanager/)  [(Reason)](https://bugzilla.redhat.com/show_bug.cgi?id=1540451)
	`systemctl stop NetworkManager && systemctl disable NetworkManager`
* Because of a bug mentioned [here](https://bugzilla.redhat.com/show_bug.cgi?id=1554922), you need to manually write the domain name in `/etc/sysconfig/network-scripts/ifcfg-ovirtmgmt` (Only in _host 1_). Write the following at the end of the file
	`DOMAIN=management.org`

## Preparing the gluster setup

### Identify the unallocated partition

* Ideally, there should be one unallocated partition. Otherwise, use `gparted` to find out.
* It should be large enough to host multiple VMs. Check the size using `fdisk` before creating the partition
* Do this carefully, otherwise, you might format the wrong partition

### Creating the partition

* Once it has been identified, use `fdisk` to create it
	* `fdisk /dev/sda`
		* Create a new partition
		* Use default values
		* Write the partition table
		* Exit
* Do `partprobe` (Let the OS know the partition table has been updated)
* The new partition is at `/dev/sda3`

### Format the partition

* Apply label to the partition (_Don't know why we use 2 commands_)
	`ssm add -p gluster /dev/sda3`
	`ssm create -p gluster --fstype xfs -n bricks`
* Create mount point of the partition
	`mkdir /gluster`
* Put the partition details in `/etc/fstab` so that the partition mounts on boot
	`echo "/dev/mapper/gluster-bricks /gluster xfs defaults 0 0" >> /etc/fstab`
* Mount the partition
	`mount -a`
* Reboot the machine and check if the partitions are mounting properly. Otherwise redo the above steps

### Add the gluster directories and start gluster

* `mkdir -p /gluster/{engine,data,iso,export}/brick`
* `systemctl start glusterd && systemctl enable glusterd`

### Edit the host files for the gluster nodes and the servers

* Edit `/etc/hosts` and add the following
	```
	10.5.20.121 user1
	10.5.20.122 user2
	10.5.20.123 user3
	10.5.20.125 management.org
	10.5.20.121 glustermount
	10.5.20.122 glustermount
	10.5.20.123 glustermount
	```

 ==**[Do the following only on the management PC]**==

## Setting up glusterfs

### Deleting a previous gluster installation

Note : You can upgrade `glusterfs` if you are doing it from scratch
```
yum update glusterfs
```
1. Stop the gluster service at all hosts
	```
	service glusterd stop
	```
3. Manually cleared the `/gluster` partition  at all hosts
	```
	rm -rf /gluster/data /gluster/engine /gluster/export /gluster/iso
	```
3. Manually clean the old gluster configurations. This and the above step might need multiple tries. You will get lot's of errors, ignore them
	```
	mv /var/lib/glusterd/glusterd.info /tmp/.
	rm -rf /var/lib/glusterd/
	mv /tmp/glusterd.info /var/lib/glusterd/.
	```
4. Reboot all the hosts (_required_)

### Creating the gluster directories

1. Stop the gluster service at all hosts
	```
	service glusterd stop
	```
2. Make bricks for gluster volumes on all 3 hosts
	```
	mkdir -p /gluster/{engine,data,iso,export}/brick
	```
3. Start the gluster service
	```
	service glusterd start
	```

### Connect to the other gluster peers

1. Connect the other hosts to *Host 1*
	```
	gluster peer probe user2
	gluster peer probe user3
	```
	You should get the following output `peer probe: success.`
	
	* Use `gluster peer status` to check if both *user2* and *user3* are in peer list of *user1*
	* If the hosts are not in `State: Peer in Cluster (Connected)` state (only in `accepting` state)
		* Detach them and redo, only then move to the next step

2. Connect the gluster bricks
	
	```
	gluster volume create engine    replica 3 arbiter 1 user1:/gluster/engine/brick user2:/gluster/engine/brick user3:/gluster/engine/brick force
	gluster volume create data      replica 3 arbiter 1 user1:/gluster/data/brick   user2:/gluster/data/brick   user3:/gluster/data/brick   force
	gluster volume create iso       replica 3 arbiter 1 user1:/gluster/iso/brick    user2:/gluster/iso/brick    user3:/gluster/iso/brick    force
	gluster volume create export    replica 3 arbiter 1 user1:/gluster/export/brick user2:/gluster/export/brick user3:/gluster/export/brick force
	```

3. Run the following commands. Ignore if they gives some error
	```
	gluster volume set engine   group virt
	gluster volume set data     group virt
	```
	
### Setting permissions

1. These following 4 commands are for giving access permissions to the gluster volumes
	```
	gluster volume set engine   storage.owner-uid 36 && gluster volume set engine   storage.owner-gid 36
	gluster volume set data     storage.owner-uid 36 && gluster volume set data     storage.owner-gid 36
	gluster volume set export   storage.owner-uid 36 && gluster volume set export   storage.owner-gid 36
	gluster volume set iso      storage.owner-uid 36 && gluster volume set iso      storage.owner-gid 36
	```
### Starting the volumes

1. Now start the gluster volumes
	```
	gluster volume start engine
	gluster volume start data
	gluster volume start export
	gluster volume start iso
	```
Now gluster is up. You can check with 
	`gluster volume status`
	`gluster peer status`

## Installing the Ovirt hosted engine

### Removing previous ovirt deployment

`ovirt-hosted-engine-cleanup`
`hosted-engine --clean-metadata --host-id=<host_id> --force-clean` (Host ID is `1,2,3`)

### Install the ovirt-engine itself

`yum install ovirt-engine-appliance -y`

### Deploy the hosted-engine
* Start `screen` (_Required_)
* Run command
	`hosted-engine --deploy`
* Gateway IP = `10.5.20.2`
* DNS = `10.5.20.121`
* Engine VM FQDN = `management.org`
* Engine VM domain name = `org`
* _Add lines for the appliance itself and for this host to /etc/hosts on the engine VM?_ Answer = No
* IP = `Static`
* Engine IP = `10.5.20.125`
* Storage type = `glusterfs`
* Path to gluster storage = `glustermount:/engine` 

At this point, all the work at the terminal is finished and the hosted engine is up. Few more steps to take

* Login to the hosted engine VM by `root@10.5.20.125`
* Put proxy in `/etc/yum.conf`
* Put proxy in `/etc/environment`
* Update the VM 
	`yum update -y`
* Install the following packages
	`yum install -y epel-release htop vim ovirt-iso-uploader`

Now we will move to the web-interface of the hosted engine, at `https://management.org/ovirt-engine`

## Setting up from the Ovirt Web Panel

### Preparing host 1

#### Add the gluster domains
* _Storage_
* _Domain_
* _[New domain]_
* Add each domain with storage type `glusterfs` and path as `glustermount:/<vol name>`

#### Setting CPU family & gluster management from ovirt
* _Compute_
* _Cluster_
* _Edit_
* _CPU type_ = `Intel SandyBridge family`
* _Enable Gluster Service_ = Check <-**[Do this step after adding the others hosts - Recommended but not mandatory]**

Our host is now ready. We can now add other hosts.

### Adding the other hosts
* _Compute_
* _Host_
* _New_
* _General_
* Give a name for the host
* Hostname = as per hostname mentioned in `/etc/hosts`. Don't give IP
* Give root password
* _Hosted engine_
* _Deploy_

**Note**
It's not tested but you need to reinstall it again if you want your other hosts to host the hosted engine. If it's successfull, then a crown icon should be visible

You are now finished with the setup

---

### Creating a VM

#### Upload ISO
* Login to management VM `ssh root@10.5.20.125`
* Download the ISO (or scp from another machine)
* Upload it to gluster domain `iso`
	`engine-iso-uploader upload -i iso <image name>` <- **[Run the command in the same directory as the .iso file]**

#### Adding the VM
* _Compute_ 
* _Virtual machine_
* _New_
* Add network interface
* Add disk
* Add iso image from `iso` gluster domain

**Note**
* For first boot, boot from CD/DVD drive to install, as the disk is empty
* After installation, boot from disk

### Access website from any PC

By default, the ovirt web portal is only accessable from the ovirt hosts. If we want to access it from outside of those machines, we need to disable a security check

* Log into hosted-engine VM
* Create the file `/etc/ovirt-engine/engine.conf.d/99-sso.conf`
* Add the following line
	`SSO_CALLBACK_PREFIX_CHECK=false`
* Restart the hosted-engine
	```
	hosted-engine --vm-shutdown
	hosted-engine --vm-start
	```

Now you should be able to login via IP from any other machine in the same LAN

[Source](http://akxc.github.io/2018/01/03/setup-ovirt-using-ovirt-node-ng-installer.html)

## Things to keep in mind and some caution

### Errors I faced
* During installation, the process got stuck at checking VM health status and eventually failed after 120 retries. Possible reasons are 
	* The VM could not start properly - Needs to be redeployed
	* The web-interface could not be reached 
	* Port `443` was blocked which is needed to check VM health. Unblock it using `iptables` (_Not sure whether this is an issue_)
* The CentOS panel was not working after deployment. This is not a big issue.
* In `/etc/hosts` at `host1`, `management.org` was getting a private IP . I'm not sure whether this is an expected behaviour. I forcefully tried to give the IP `10.5.20.125` during installation. It did not work. (_Don't worry about this. It's a temporary assignment. Don't modifying anything during installation_)
* Error `HTTP respose code is 400` = make sure all the gluster hosts are up before deploying
* It's very unlikely but after a failed installation, somehow my hosted-engine VM got installed. It's better to reinstall again

### Tips
* Before deploying, stop the network manager [(Source)](https://bugzilla.redhat.com/show_bug.cgi?id=1540451)
* In case of some connection issues, flush `iptables`
	`iptables -F`
* Make sure all entries in `/etc/hosts` and `/etc/resolve` are correct 
* Make sure `dnsmasq` is running
* When not working on the system, set it into global maintainance mode

#### hosted-engine re-deploying caution
* your interface has changed from `p8p1` to `ovirtmgmt`
* you need to reinstall glusterfs from scratch (mentioned below) if a hosted engine had been previously been successfully installed since it's still associated with the previous hosted-engine **(Important)**
* don't format the partition, `rm -rf` will do
	* I used [this resource](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/mdatarecover.html) to rescue my server. [Another source](https://www.centos.org/forums/viewtopic.php?f=47&t=50064&sid=6fe48e4f66e012d134b887ede16487f7&start=10)
* check RAM usage using `htop` to find whether hosted-engine has been properly removed or not
* keep backup of the file `/var/lib/glusterd/groups/virt` 
	* If this vanishes, gluster won't be registered with ovirt due to permission issues
	* You probably need to reinstall gluster at this point
		* [(source^1^)](https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.0/html/Quick_Start_Guide/chap-Quick_Start_Guide-Virtual_Preparation.html) The concept of permission
		* [(source^2^)](https://github.com/gluster/glusterfs/blob/master/extras/group-virt.example) Sample file
		* [(source^3^)](https://bugzilla.redhat.com/show_bug.cgi?id=902644) 
		* [(source^4^ with solution)](https://bugzilla.redhat.com/show_bug.cgi?id=1129592)

#### uninstalling glusterfs
* Remove all gluster packages
`yum remove -y ovirt-hosted-engine-setup glusterfs-server vdsm-gluster glusterfs`
* Install the above
* During reinstalling of glusterfs, you will need to install `ovirt-engine-appliance` again at _host 1_

#### Web interface caution
* In case there was a failure while connecting a host (not the primary host), remove it
	* `ovirt-hosted-engine-cleanup` - Run this in the failed host
	* Set the host in **maintainance mode** (_Mandatory step_)
	* Remove the host
* Don't connect gluster service before connecting the hosts
	* If anything goes while connecting them, you cannot delete the hosts 
	* Now you have to redeploy
* Give correct CPU family in cluster. Otherwise hosts will not activate (_Probably, but not 100% sure_)
