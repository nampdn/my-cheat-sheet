https://blog.zerosector.io/2018/07/28/kvm-qemu-windows-10-gpu-passthrough/

https://heiko-sieger.info/running-windows-10-on-linux-using-kvm-with-vga-passthrough/#About_keyboard_and_mouse

# Deploy juju-controller in lxd before using manual cloud

## https://blog.simos.info/how-to-make-your-lxd-containers-get-ip-addresses-from-your-lan-using-a-bridge/

Creating a new profile in LXD for bridge networking
In LXD, there is a default profile and then you can create additional profile that either are independent from the default (like in the macvlan post), or can be chained with the default profile. Now we see the latter.

First, get a list of all available existing profiles. There is a single profile, the default one from LXD, and is used by 11 LXD containers. This means that this LXD installation has 11 containers.
```
$ lxc profile list
+---------------+---------+
| NAME          | USED BY |
+---------------+---------+
| default       | 11      |
+---------------+---------+
```
Then, create a new and empty LXD profile, called bridgeprofile.

```
$ lxc profile create bridgeprofile
Here is the fragment to add to the new profile. The eth0 is the interface name in the container, so for the Ubuntu containers it does not change. Then, bridge0 is the interface that was created by NetworkManager. If you created that bridge by some other way, add here the appropriate interface name. The EOF at the end is just a marker when we copy and past to the profile.

description: Bridged networking LXD profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: bridge0
    type: nic
EOF
```
Paste the fragment to the new profile.

$ cat <<EOF | lxc profile edit bridgeprofile
(paste here the full fragment from earlier)
The end result should look like the following.
```
$ lxc profile show bridgeprofile
config: {}
description: Bridged networking LXD profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: bridge0
    type: nic
name: bridgeprofile
used_by:
```

Now, list again the profiles so that we can verify the newly created profile, bridgeprofile. It is there, and it is not used yet by a LXD (lex-dee) container.

```
$ lxc profile list
+---------------+---------+
| NAME          | USED BY |
+---------------+---------+
| bridgeprofile | 0       |
+---------------+---------+
| default       | 11      |
+---------------+---------+
```

If it got messed up, delete the profile and start over again. Here is the command.

```
$ lxc profile delete profile_name_to_delete
```

Creating containers with the bridge profile
Now we are ready to create a new container that will use the bridge. We need to specify first the default profile, then the new profile. This is because the new profile will overwrite the network settings of the default profile.

```
$ lxc launch -p default -p bridgeprofile ubuntu:x mybridge
```

Creating mybridgeStarting mybridge
Here is the result.

```
$ lxc list
+-------------+---------+---------------------+------+
| mytest | RUNNING | 192.168.1.72 (eth0)      |      |
+-------------+---------+---------------------+------+
| ...                                         | ...  |
```

The container mybridge is accessible from the local network.

# Fix broken machine:

## https://askubuntu.com/questions/433842/how-to-resolve-error-machine-is-already-provisioned-in-manual-provision-set-up

# LXD Default Profile (Bridge)

```yaml
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: br1
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
```

# SYSCTL LIMIT
Edit file
`sudo vim /etc/sysctl.conf`

```
fs.inotify.max_queued_events = 1048576
fs.inotify.max_user_instances = 1048576
fs.inotify.max_user_watches = 1048576
```

CLEAN UP JUJU AGENT
```bash
sudo rm -rf /var/lib/juju
sudo rm -rf /lib/systemd/system/juju*
sudo rm -rf /run/systemd/units/invocation:juju*
sudo rm -rf /etc/systemd/system/juju*
```

# LXD Ceph Profile
```bash
# snap install lxd
# lxc profile create ceph
# lxc profile edit ceph
```

## Config `ceph` profile
```yaml
config:
  user.user-data: |
    #cloud-config
    ssh_pwauth: yes

    users:
     - name: ubuntu
       passwd: "$6$iBF0eT1/6UPE2u$V66Rk2BMkR09pHTzW2F.4GHYp3Mb8eu81Sy9srZf5sVzHRNpHP99JhdXEVeN0nvjxXVmoA6lcVEhOOqWEd3Wm0"
       lock_passwd: false
       groups: lxd
       shell: /bin/bash
       sudo: ALL=(ALL) NOPASSWD:ALL


    runcmd:
     - mount -t 9p config /mnt/
     - cd /mnt && ./install.sh
     - reboot
description: "Ceph Virtual Machine"
devices:
  config:
    source: cloud-init:config
    type: disk
name: ceph
```

# Disk Pool

Check disk label:

```
udevadm info --query=all --name=/dev/sda | grep ID_SERIAL
```

## LXD Storage
```bash
zpool create -f hdd-1 /dev/sda
lxc storage create hdd-1 zfs source=hdd-1
lxc storage set hdd-1 size 5529GB
lxc storage volume create hdd-1 hdd-disk --type=block
lxc storage volume set hdd-1 hdd-disk size 5529GB
```

## LXD btrfs pool
```
export DISK_POOL=hdd-1
export DISK_SIZE=465GB
lxc storage create $DISK_POOL btrfs source=/dev/sdX && \
lxc storage set $DISK_POOL size $DISK_SIZE && \
lxc storage volume create $DISK_POOL hdd-disk --type=block && \
lxc storage volume set $DISK_POOL hdd-disk size $DISK_SIZE && \
lxc storage volume show $DISK_POOL hdd-disk
```

# LXD VM
```bash
export VM_NAME=ceph-osd-06-ctl
lxc init ubuntu:18.04 $VM_NAME -p default -p ceph --vm
lxc config set $VM_NAME limits.memory 6GB # Set to 1GB memory per 1TB storage, skip if nvme/ssd < 1TB
lxc storage volume attach $DISK_POOL hdd-disk $VM_NAME sdb /dev/sdb
lxc start $VM_NAME && \
lxc console $VM_NAME => ctrl+a q
lxc shell $VM_NAME
```

# Edit netplan static (if manuall ip)
```yaml
network:
    ethernets:
        enp5s0:
            dhcp4: false
            addresses: [10.0.1.3/16]
            gateway4: 10.0.0.1
            nameservers:
                    addresses: [10.0.0.1, 1.1.1.1, 1.0.0.1]


=>> DO NOT EDIT GENERATED MACADDRESS
            match:
                macaddress: 00:16:3e:1c:91:93
            set-name: enp5s0
    version: 2
```

## Add new machine to juju
```
ssh-copy-id ubuntu@10.0.1.3
juju add-machine ssh:ubuntu@10.0.2.2
```

# Create Ceph Mon lxd
```
lxc launch ubuntu:18.04 ceph-mon-01
```

# Ceph Mon Container Network
```yaml
network:
    version: 2
    ethernets:
        eth0:
                dhcp4: false
                addresses: [10.0.1.22/16]
                gateway4: 10.0.0.1
                nameservers:
                        addresses: [10.0.0.1, 1.1.1.1, 1.0.0.1]
```

# START

```# juju add-cloud```

Enter the ssh connection string for controller, username@<hostname or IP> or <hostname or IP>: root@10.0.0.2

```# juju bootstrap vgm manual-controller```


```# juju add-model openstack```


## Optimize Ceph with OpenStack

1. [The Dos and Don'ts for Ceph for OpenStack](https://ceph.io/planet/the-dos-and-donts-for-ceph-for-openstack/)
2. [Expanding Ceph clusters with Juju](https://zeestrataca.com/posts/expanding-ceph-clusters-with-juju/)
3. [Using libvirt with Ceph](https://documentation.suse.com/ses/6/html/ses-all/cha-ceph-libvirt.html)
4. [Ceph OSD Journal](https://docs.ceph.com/docs/master/rados/configuration/osd-config-ref/)
5. [HOW-TO How to integrate Ceph with OpenStack](https://superuser.openstack.org/articles/ceph-as-storage-for-openstack/)
6. [Charm Ceph Mon](https://opendev.org/openstack/charm-ceph-mon)
7. [Openstack Juju Official Document](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/ussuri/install-openstack.html)
8. [Ceph CRUSHMAP](https://docs.ceph.com/docs/mimic/rados/operations/crush-map/)
9. [Ceph libvirt](https://docs.ceph.com/docs/master/rbd/libvirt/)


## Miscellanous

### Allow Ceph pool deletion: 
```
ceph tell mon.\* injectargs '--mon-allow-pool-delete=true'
ceph osd pool rm test-pool test-pool --yes-i-really-really-mean-it
```
### Wipe a physical disk when something went wrong.

Usually, when using `lxc storage create pool1 btrfs source=/dev/sdX` it will indicate noisy message, we can easly wipe all the thing on that disk.

```
wipefs -a /dev/sdb
```

## Ceph

### Authorizing Ceph Client

### Ceph Pool

### RBD QEMU

### RBD SNAPSHOT
