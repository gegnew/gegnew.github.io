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

###Parts list
- 4-8 Raspberry Pi 4 (for the compute nodes)
- 1x Raspberry Pi 4 (for the master/login node)
- 5-9 MicroSD Cards (Raspi4 already comes with a 16Gb card, but we probably need more storage)
- 5-9 micro-USB power cables (comes with Raspi 4)
- 1x network switch (10/100/100 Mbps, modular?)
- 10-port USB power supply
- 1 big USB hard drive (or a NAS box?) for shared storage


## Step 1: Flash the Pi's
Flash the SD cards with Raspian Lite (or Arch? Whatever). 
Before unmounting, we need to enable SSH. @glmdev says to open /boot and create an empty file called "ssh". @kubecloud says to make a `/Volumes/boot/ssh` file
Interestingly, then @kubecloud says to `ssh pi@raspberrypi.local` rather then finding the ip as below

## Step 2: Set up the network
### Option 1: @glmdev
We need to assign static IP addresses to the nodes. Plug the switch into the network and connect a Pi to the switch. 
Then use `nmap` or `tshark` to find the Pi's IP address. For me, I think this will be `tshark -Y "ip.src==192.168.0.0/16 and ip.dst==192.168.0.0/16"`
SSH into the pi with `ssh pi@ip.addr.here`. The default password is "raspberry".

### Option 2: @kubecloud
SSH to pi with `ssh pi@raspberrypi.local`. Run the script below with `sh hostname_and_ip.sh [new hostname] [new static IP] [router IP]`. This kinda seems easier.
        ~~~~
        #!/bin/sh

        hostname=$1
        ip=$2 # should be of format: 192.168.1.100
        dns=$3 # should be of format: 192.168.1.1

        # Change the hostname
        sudo hostnamectl --transient set-hostname $hostname
        sudo hostnamectl --static set-hostname $hostname
        sudo hostnamectl --pretty set-hostname $hostname
        sudo sed -i s/raspberrypi/$hostname/g /etc/hosts

        # Set the static ip

        sudo cat <<EOT >> /etc/dhcpcd.conf
        interface eth0
        static ip_address=$ip/24
        static routers=$dns
        static domain_name_servers=$dns
        EOT
        ~~~~
@kubecloud sets the IPs as 192.168.1.100 for master, then X.101... for the nodes. Then reboot; you should be able to ssh to the pi like `ssh pi@[new hostname].local`

### Option 1: @glmdev, doing it with SLURM
## Step 3: Set up the Pi's
Basic config: `sudo raspi-config`. Set the default password, locale, timezone, and expand the filesystem. Exit the setup utility.
Set the hostname in some sort of sane fashion:
~~~~
sudo hostname node01    # make these names serial
sudo vim /etc/hostname  # change the hostname here
sudo vim /etc/hosts     # and here
~~~~

We need to make sure that the system time is correct. `ntpdate` will periodically sync the system time in the background. Install with `sudo apt install ntpdate -y` and reboot.

## Step 4: Shared storage
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

## Step 5: Configure the master node
Most clusters use a scheduler, which will accept jobs and run them on the next available set of nodes. We will use SLURM as our scheduler on the master node.

SSH into the master node and edit the `/etc/hosts` file. Add lines like below, which are the ip addresses and hostnames of the nodes, which will make resolution easier.
~~~~
\<ip addr of node02>    node02
\<ip addr of node03>    node03
\<ip addr of node03>    node03
~~~~

Install the SLURM controller packages, `slurm-wlm`. The default SLURM config is in `/usr/share/doc/slurm-client/examples/slurm.conf.simple.gz`; copy it to `/etc/slurm-llnl` and unzip it with `gzip -d`. Rename it with `mv slurm.conf.simple slurm.conf`. You should have a file:`/etc/slurm-llnl/slurm.conf`. Modify the first two configuration lines to include the master node hostname and IP:
~~~~
ControlMachine=node01
ControlAddr=\<node01 ip addr>
~~~~

SLURM can allocate resources in a variety of ways (check SLURM documentation), but we'll use the "consumable resources" method. To do this, edit the `SelectType` field to:
~~~~
SelectType=select/cons\_res
SelectTypeParameters=CR\_CPU
~~~~

In the "LOGGING AND ACCOUNTING" section you can change the ClusterName parameter. Give your cluster a name.

Add the nodes to the SLURM config by deleting the example entry (near the end of the config file) and replacing it with your nodes:
~~~~
NodeName=node01 NodeAddr=\<ip addr node01> CPUs=4 State=UNKNOWN
NodeName=node02 NodeAddr=\<ip addr node02> CPUs=4 State=UNKNOWN
NodeName=node03 NodeAddr=\<ip addr node03> CPUs=4 State=UNKNOWN
NodeName=node04 NodeAddr=\<ip addr node04> CPUs=4 State=UNKNOWN
~~~~
Apparently, this includes the master node.

SLURM runs jobs on partitions, i.e. groups of nodes, so we can set a default partition with all of our compute nodes on it. Delete the example partition in the file, then add the following:
`PartitionName=mycluster Nodes=node[02-04] Default=YES MaxTime=INFINITE State=UP`

# Option 2: Kubernetes
Kubernetes tutorials (the first one largely copped below):
[nycdev](https://medium.com/nycdev/k8s-on-pi-9cc14843d43)
[@kubecloud](https://kubecloud.io/setting-up-a-kubernetes-1-11-raspberry-pi-cluster-using-kubeadm-952bbda329c8)
[ITNEXT](https://itnext.io/building-a-kubernetes-cluster-on-raspberry-pi-and-low-end-equipment-part-1-a768359fbba3)

After pi setup, install docker and set permissions:
~~~~
curl -sSL get.docker.com | sh && \
sudo usermod pi -aG docker && \
newgrp docker
~~~~

Kubernetes needs [swap disabled](https://github.com/kubernetes/kubernetes/pull/55399):
~~~~
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo update-rc.d dphys-swapfile remove
~~~~
Then `sudo swapon --summary` should return empty

[nycdev](https://medium.com/nycdev/k8s-on-pi-9cc14843d43) says to add the following line to the `/boot/cmdline.txt` file, in the same line as all the other text,  but the @kubecloud tutorial does not. ITNEXT also says to do this.
`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
Then reboot.

SSH in again and edit `/etc/apt/sources.list.d/kubernetes.list`, adding `deb http://apt.kubernetes.io/ kubernetes-xenial main` to the file.
Then add the key:
`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

Update the repo with `sudo apt-get update`.

Install `kubeadm`, which will also install `kubectl`:
`sudo apt-get install -qy kubeadm`


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

### Worker node setup:
Run `kubeadm token create --print-join-command` to get the join token. 
You will see the join-token, which you need for other nodes to join the network. It will look like this:
`kubeadm join --token 9e700f.7dc97f5e3a45c9e5 192.168.0.27:6443 --discovery-token-ca-cert-hash sha256:95cbb9ee5536aa61ec0239d6edd8598af68758308d0a0425848ae1af28859bea`
Run this command on each node and check the status by running `kubectl get nodes` (or maybe get node). You will note that the status is "NotReady", as we have not yet set up container networking.

In case that doesn't work, `kubeadm token list` will get the available tokens and the following will get the sha has:
`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`

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


# Deploying a test app on your Kubernetes cluster


### Other thoughts from other tutorials:
Kubernetes only requires that you enable an official apt repostitory (from apt.kubernetes.io) and install kubelet, kubeadm, docker, and kubectl(only on master)
