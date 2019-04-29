# What

This will provision a Swarm/PX cluster on a local VirtualBox. A Docker container running etcd will also be provisoned on the master for PX.

# How

1. Install [Vagrant](https://www.vagrantup.com/downloads.html) and [VirtualBox](https://www.virtualbox.org/wiki/Downloads).
2. Clone this repo and cd to it.
3. Edit top section of `Vagrantfile` as necessary.
4. Edit CentOS-Base.repo as necessary.
5. Generate SSH keys:
```
$ ssh-keygen -t rsa -b 2048 -f id_rsa
```
This will allow SSH as root between the various nodes.

6. Start the master VM:
```
vagrant up master
```

7. Start the workers:
```
$ vagrant up
```

8. Check the status of the Portworx cluster:
```
$ vagrant ssh node1
[vagrant@node1 ~]$ sudo /opt/pwx/bin/pxctl status
Status: PX is operational
License: Trial (expires in 31 days)
Node ID: 1d0cf4f2-9fe8-4e6d-b1c3-816cf3440f53
	IP: 192.168.99.101
 	Local Storage Pool: 1 pool
	POOL	IO_PRIORITY	RAID_LEVEL	USABLE	USED	STATUS	ZONE	REGION
	0	LOW		raid0		20 GiB	3.0 GiB	Online	default	default
	Local Storage Devices: 1 device
	Device	Path		Media Type		Size		Last-Scan
	0:1	/dev/sdb	STORAGE_MEDIUM_MAGNETIC	20 GiB		29 Apr 19 13:52 UTC
	total			-			20 GiB
Cluster Summary
	Cluster ID: px-test-cluster
	Cluster UUID: ef166d67-8a64-47e7-a251-ebe1fb7e21e4
	Scheduler: none
	Nodes: 3 node(s) with storage (3 online)
	IP		ID					SchedulerNodeName			StorageNode	Used	Capacity	Status	StorageStatus	Version		Kernel				OS
	192.168.99.103	91c35715-85a6-4729-9831-fac00eb23bef	91c35715-85a6-4729-9831-fac00eb23bef	Yes		0 B	20 GiB		Online	Up		2.0.3.4-0c0bbe4	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
	192.168.99.102	62c5477a-81d8-405f-a819-0b1788424ab7	62c5477a-81d8-405f-a819-0b1788424ab7	Yes		0 B	20 GiB		Online	Up		2.0.3.4-0c0bbe4	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
	192.168.99.101	1d0cf4f2-9fe8-4e6d-b1c3-816cf3440f53	1d0cf4f2-9fe8-4e6d-b1c3-816cf3440f53	Yes		3.0 GiB	20 GiB		Online	Up (This node)	2.0.3.4-0c0bbe4	3.10.0-957.5.1.el7.x86_64	CentOS Linux 7 (Core)
Global Storage Pool
	Total Used    	:  3.0 GiB
	Total Capacity	:  60 GiB
```

# Notes

The object is to provision everything on top of a base CentOS installation to make it easier to test different software versions. This is inherently slower than baking everything into a prebuilt image.

After each VM is provisioned, the bootstrap process runs in the background (so Vagrant can continue provisioning the subsequent nodes in parallel).

The process logs to `/var/log/vagrant.boostrap` on each node. When the process is completed, the line "End" is logged.

A Docker registry cache will run on the master node, and worker nodes will pull all of their docker.io images via the proxy to minimise bandwidth usage.
