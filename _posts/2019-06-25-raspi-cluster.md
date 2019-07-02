---
layout: post
title: Building a Raspberry Pi Cluster
categories: [Raspberry Pi, distributed computing]
---

Johnny (my roommate) and I were quite excited when the [Raspberry Pi 4](https://www.raspberrypi.org/blog/raspberry-pi-4-on-sale-now-from-35/) released this week. Johnny has been talking about building a cluster with Pi's, as a means of learning about cluster orchestration, continuous integration, load balancing / autoscaling, and distributed data processing. As I usually do, I'm keen to hitch myself to Johnny's car and learn a little bit about how this stuff works. We're just about to order our materials, but I've decided to do some research into what this project will entail.

# Getting my bearings
I'm copping this completely from [@glmdev's guide](https://medium.com/@glmdev/building-a-raspberry-pi-cluster-784f0df9afbd) on this. I'll run through it a second time and compare it to [this other guide](http://www.raspberrywebserver.com/raspberrypicluster/raspberry-pi-cluster.html). 
I actually am running through a second time based on [this kubernetes pi cluster tutorial](https://kubecloud.io/setting-up-a-kubernetes-1-11-raspberry-pi-cluster-using-kubeadm-952bbda329c8).
Some other useful reading (that I haven't read yet) is [this Kubernetes blog post](https://kubernetes.io/blog/2018/03/apache-spark-23-with-native-kubernetes/) about running Apache Spark on Kubernetes and and [this Apache doc page](https://spark.apache.org/docs/2.3.0/running-on-kubernetes.html) on running Spark on Kubernetes.

###Parts list (get links from Johnny)
- 4-8 Raspberry Pi 4 (for the compute nodes)
- 1x Raspberry Pi 4 (for the master/login node)
- 5-9 MicroSD Cards (Raspi4 already comes with a 16Gb card, but we probably need more storage)
- 5-9 micro-USB power cables (comes with Raspi 4)
- 1x network switch (10/100/100 Mbps, modular?)
- 10-port USB power supply
- 1 big USB hard drive (or a NAS box?) for shared storage


## Step 1: Set up the Raspberry Pi's
For fun, we decided to run the master and two worker nodes on Raspbian Lite and one worker node on Arch ARM. Setup for each as follows:

### Raspbian
    1. download the Raspbian image from [here](https://www.raspberrypi.org/downloads/raspbian/)
    2. verify the download by running `sha256sum [raspian_image.zip]` and comparing it to the hash on the downloads page
    3. `unzip` the image file
    4. insert sd card into computer
    5. locate device with `lsblk` or `sudo fdisk -l`
    6. ensure partitions are unmounted with `sudo fdisk /dev/sdX*`
    7. copy the image file to the sd card with `sudo dd bs=4M status=progress if=[raspian_iso] of=/dev/sdX` 

### Arch ARM
Note that this is NOT the officially supported Arch Linux distribution, but one redesigned for ARM architectures.
Follow the instructions [here](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3) under the "Installation" tab.
After that, I completed the install using [this tutorial](https://elinux.org/ArchLinux_Install_Guide), except for the swap file section, where I did:

TODO: finish this setup
#DONT DO THIS

    Set the swap file:
        ~~~~
        fallocate -l 1024M /swapfile
        chmod 600 /swapfile
        mkswap /swapfile
        swapon /swapfile
        echo 'vm.swappiness=1' > /etc/sysctl.d/99-sysctl.conf
        ~~~~

    8. Add color to pacman with `sed -i 's/#Color/Color/' /etc/pacman.conf`


### Configure the Pi's
I tried to enable ssh by placing an empty file called "ssh" in the /boot directory. This did not seem to work at all, so I plugged the Pi's into a monitor and used `sudo raspi-config`.
Set the hostname, enable ssh, expand the filesystem, set the password, locale, and timezone.
We need to make sure that the system time is correct. `ntpdate` will periodically sync the system time in the background. Install with `sudo apt install ntpdate -y` and reboot.

### Set up the network
We did not set statis IPs for the Pi's, as we weren't worried about the IP leases being reassigned on our low-traffic local network. We might do this later.

## Step 2: Set up Kubernetes
Kubernetes tutorials (the first one largely copped below):
[nycdev](https://medium.com/nycdev/k8s-on-pi-9cc14843d43)
[@kubecloud](https://kubecloud.io/setting-up-a-kubernetes-1-11-raspberry-pi-cluster-using-kubeadm-952bbda329c8)
[ITNEXT](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3)

After pi setup, [install docker](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) and set permissions:
`sudo apt-get update`
`apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common`

Import the Docker GPG key:
`curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -`

`add-apt-repository` wasn't working, so I added the following in my `/etc/apt/sources.list`
`deb [arch=amd64] https://download.docker.com/linux/debian stretch stable`

Update again, then install Docker:
`curl -sSL get.docker.com | sh`

When we were working on this (June 30 2018), there was [an error](https://github.com/docker/for-linux/issues/709) in the docker package numbering (?) for debian distros. A helpful comment from @caio resulted in this solution for the curl command:
~~~~
sudo rm /etc/apt/sources.list.d/docker.list;
curl -sL get.docker.com | sed 's/9)/10)/' | sh
~~~~

`sudo usermod -aG docker $USER`

Kubernetes needs [swap disabled](https://github.com/kubernetes/kubernetes/pull/55399):
~~~~
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo update-rc.d dphys-swapfile remove
~~~~
Then `sudo swapon --summary` should return empty

For some reason, no guides we found suggested disabling `dphys-swapfile` with systemd, but we found it to be necessary:
`sudo systemctl disable dphys-swapfile`

Add the following line to the `/boot/cmdline.txt` file, in the same line as all the other text
`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
For safety, `sudo cp /boot/cmdline.txt /boot/cmdline_backup.txt`.

Add the GPG key: 
`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`
Note that if you have to run this with sudo, you need a sudo on both sides of the pipe.

Then do:
`echo deb http://apt.kubernetes.io/ kubernetes-xenial main | sudo tee /etc/apt/sources.list.d/kubernetes.list`
Then `sudo apt-get update`.

Install kubeadm, which will also install kubectl, with `sudo apt-get install -y kubeadm` and reboot.

That is the common setup. 

### Master node setup:
Next, on only the master node:

Pull images:
`sudo kubeadm config images pull -v3`

Use Weave Net as a network overlay:
`sudo kubeadm init --token-ttl=0`
This sets our token to never expire, which SHOULD NOT be done in production.
Alternatively, ITNEXT suggests initializing kubernetes with the `--pod-network-cidr=10.244.0.0/16` flag (and no `sudo`, for some reason). I need to look into what that flag does.

`kubeadm` pulls docker images for kubernetes master components, generates [PKI certificates](https://kubernetes.io/docs/setup/certificates/) and starts services. It returns a snippet of code to run after, like 15 minutes:
~~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~~

You can check that this worked with `kubectl get node`.

### Install ingress controller (worker node)
Run `kubeadm token create --print-join-command` to get the join token. 
You will see the join-token, which you need for other nodes to join the network. It will look like this:
`kubeadm join --token 9e700f.7dc97f5e3a45c9e5 192.168.0.27:6443 --discovery-token-ca-cert-hash sha256:95cbb9ee5536aa61ec0239d6edd8598af68758308d0a0425848ae1af28859bea`
Run this command on each node and check the status by running `kubectl get nodes` (or maybe get node). You will note that the status is "NotReady", as we have not yet set up container networking.

In case that doesn't work, `kubeadm token list` will get the available tokens and the following will get the sha has:
`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`

At this point, we ran into a problem with Docker not starting due to some problem with [dual config files](https://github.com/moby/moby/issues/34104). Johnny fixed it by adding the following line in `/lib/systemd/system/docker.service` to use systemd:
`ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd`

I solved it by changing the ExecStart statment to be `ExecStart=/usr/bin/dockerd`. After this, I got an error from `kubeadm join` suggesting I follow [this guide](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) to change the Docker cgroup driver from cgroupfs to systemd.
Actually, as of 1am on 7/2, I haven't actually fixed this, but I did give up on the node I was working on.



Install the Weave Net network driver:
    `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
    On the master AND all the workers run:
    `sudo sysctl net.bridge.bridge-nf-call-iptables=1`

ITNEXT suggests using [flannel](https://github.com/coreos/flannel) instead, but same idea:
`kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml`. Maybe check the latest version of flannel before running this.


### Additional steps:
We can access the cluster, but only while SSHed to the master node. To access it from our local machine, do:
`scp pi@x.x.x.100:.kube/config .`
to copy the config file from the master node to your computer

If you have kubectl on your local machine, you can set `kubeconfig` up by either a) overriding the `config` file in `$HOME/.kube/config` or add the new config file on top of it by:
`export KUBECONFIG=<location to config from pi>:$HOME/.kube/config`
More info on this [here](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

# Internet Gateways
The [OSI model](https://en.wikipedia.org/wiki/OSI_model#Layer_1:_Physical_Layer) is a conceptual model of a computing system. From bottom to top:
    1. Physical Layer - Responsible for transmission of data between a device and a transimssion medium. 
    2. Data Link Layer - Provides node-to-node data transfer.
    3. Network Layer - Provides means of transferring data ("packets") from one node to another connected in different networks. 
    4. Transport Layer - Provides means of transferring data from a source to a destination host.
    5. Session Layer - Controls the connections between computers, establishing, managing, and terminating connections between local and remote application.
    6. Presentation Layer - Establishes context between application-layer entities.
    7. Application Layer - This layer and the user both interact directly with the software application. Communication component.
    
An internet gateway is a hardware/software system intended to provide network connectivity from the public internet to a private network at OSI Level 4. This can be done wtih [Network address translation (NAT)](https://en.wikipedia.org/wiki/Network_address_translation) or various proxy servers (i.e. HAProxy, nginx). I have no idea what any of this means.


# Deploying a test app on your Kubernetes cluster

Install [minikube](https://kubernetes.io/docs/setup/minikube/) locally to test your setup before deploying to your cluster.








## Shared storage
We will set up freddie as a network filesystem (NFS) later, but she isn't hooked up right now, so we'll save it for later.

It will go as follows:

For the cluster to be a cluster, a job should be able to run on any node; therefore every node must access the same files. We can use a hard drive connected to the master node and then export the drive as a network file system (NFS). We could use freddie as our master node and our shared file system, if we feel like it. Alternatively, we could use a NAS box as our NFS, by exporting an NFS share from the NAS and mounting it on the nodes. I'll assume we're using a Pi the master node and an external HD as our shared storage.

SSH into the master node. Connect the shared storage drive and format as ext4: `sudo mkfs.ext4 /dev/sda1` or whatever it might be.
Create a directory to mount the drive to, such as "clusterfs".
~~~~
sudo mkdir /clusterfs
sudo chown nobody.nogroup -R /clusterfs
sudo chmod 777 -R /clusterfs
~~~~

We want this drive to mount automatically on boot, so find the shared storage UUID with `blkid`. Add it to `/etc/fstab` as `UUID=[uuid] /clusterfs ext4 defaults 0 2`. Finally, mount the drive.

Next, we'll set permissions. Johnny and I will have to discuss what these ought to be, but loose permissions would be set as:
~~~~
sudo chown nobody.nogroup -R /clusterfs
sudo chmod -R 766 /clusterfs
~~~~

On the master node, install the NFS server with `sudo apt install nfs-kernel-server -y`. To export the NFS share, edit `/etc/exports` and add the line:
`/clusterfs <ip addr>(rw, sync, no_root_squash, no_subtree_check)`, replacing "\<ip addr\>" with the IP address schema from your local network. (i.e. 192.168.1.X)
This gives the client read-write access, `sync` forces changes to be written on each transation, `no_root_squash` enables the root users of clients to write files as root, and `no_subtree_check` prevents errors caused by a file being changed while another system is using it.

On all the other nodes, install `nfs-common` instead of `nfs-kernel-server`. Create a mount directory (i.e. /clusterfs) and set permissions. Try `chmod 777` instead of `766`. Set up automatic mounting by adding the following line to your fstab:
`<master node ip>:/clusterfs    /clusterfs  nfs defaults    0 0`
Mount the drive. Test it by creating a file in /clusterfs on one node and check to see if it shows up on the other nodes.

After setting up k8s, [this doc page](https://docs.docker.com/ee/ucp/kubernetes/storage/use-nfs-volumes/) and [this tutorial](https://matthewpalmer.net/kubernetes-app-developer/articles/kubernetes-volumes-example-nfs-persistent-volume.html) should be helpful.
