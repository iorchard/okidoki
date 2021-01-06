Install HA Cluster from scratch
=================================

This is a guide to install HA cluster from scratch.

I assume there are 3 controllers and 3 workers.

* controllers: taco2-ctrl{1,2,3} (192.168.21.4{1,2,3})
* workers: taco2-comp{1,2,3}     (192.168.21.4{4,5,6})

All machines are CentOS 7.8.

Set up Corosync and Pacemaker on controllers
---------------------------------------------

Install the HA cluster software on all controllers.::

   $ sudo yum install -y pacemaker pcs psmisc policycoreutils-python

Enable pcs daemon on all controllers.::

   $ sudo systemctl start pcsd.service
   $ sudo systemctl enable pcsd.service

Set a password for hacluster user on all controllers.::

   $ sudo passwd hacluster

Authenticate all controllers on the first controller (taco2-ctrl1).::

   $ sudo pcs cluster auth taco2-ctrl{1,2,3}
   Username: hacluster
   Password:

Generate and synchronize the HA cluster configuration
(/etc/corosync/corosync.conf) and pacemaker authentication key 
(/etc/pacemaker/authkey) on the first controller.::

   $ sudo pcs cluster setup --name okidoki taco2-ctrl{1,2,3}

Start the HA cluster.::

   $ sudo pcs cluster start --all

Disable stonith to avoid having to cover fencing device configuration.::

   $ sudo pcs property set stonith-enabled=false

See the current HA cluster status.::

   $ sudo pcs status cluster


Set up pacemaker-remote on workers
--------------------------------------

Install the HA cluster software on all workers.::

   $ sudo yum install -y pacemaker-remote resource-agents pcs

Create a directory to save the shared authentication key on all workers.::

   $ sudo mkdir -p --mode=0750 /etc/pacemaker
   $ sudo chgrp haclient /etc/pacemaker

Copy the authentication key from one of controllers on all workers.::

   $ ssh clex@<one_of_controllers> sudo cat /etc/pacemaker/authkey | \
      sudo tee /etc/pacemaker/authkey
   $ sudo chmod 0400 /etc/pacemaker/authkey
   $ sudo chown hacluster:haclient /etc/pacemaker/authkey

Enable pcs daemon on all workers.::

   $ sudo systemctl start pcsd.service
   $ sudo systemctl enable pcsd.service

Set a password for hacluster user on all workers.::

   $ sudo passwd hacluster

Authenticate all workers on the first controller (taco2-ctrl1).::

   $ sudo pcs cluster auth taco2-comp{1,2,3}
   Username: hacluster
   Password:

Add all workers as remote nodes on one of controllers.::

   $ sudo pcs cluster node add-remote <worker_hostname>


See the cluster status.::

   $ sudo pcs status

There should be 3 controllers in Online and 3 workers in RemoteOnline.

