Install ssh fence agent
=========================

It is not recommended to use ssh for fence agent.
It's just a test for how fencing is working.

Never use it in the production HA cluster.

Pre-requisites
---------------

HA cluster should be set up and running.

The package sshpass should be installed on all worker nodes.

There should be at least two networks - HA cluster network and fencing network

* HA Cluster network: corosync membership network
* Fencing network: the network to use for fencing (ssh, IPMI, etc.)

Install
----------

Copy ssh fence agent file `fence_ssh` to all controller nodes.::

   $ cat hacluster/fence_ssh | ssh -T <controller_node> \
      "cat|sudo tee /usr/sbin/fence_ssh && sudo chmod 0755 /usr/sbin/fence_ssh"

Create stonith resources for all worker nodes in pacemaker.::

   $ read -s -p 'user pw: ' USERPW
   $ export USERPW
   $ sudo -E pcs stonith create stonith-<worker_hostname> \
      fence_ssh user=<username> hostname=<worker_ip> \
      password=$USERPW sudo=true pcmk_host_list="<worker_hostname>"
   $ sudo pcs constraint location add stonith-<worker_hostname>-loc \
      stonith-<worker_hostname> <worker_hostname> INFINITY

USERPW should be the <username>'s password on each worker node.

Replace worker_hostname and username appropriately.

<worker_ip> should be the ip address of fencing network.

Enable stonith and set stonith-action=off to power off the node instead
of rebooting.::

   $ sudo pcs property set stonith-enabled=true
   $ sudo pcs property set stonith-action=off

See the cluster status. It shows stonith resources.

Test fencing
--------------

To test fencing, cut the HA cluster network connection in one of worker nodes.

Let's say worker1's HA cluster network interface is eth0.
Cut the HA cluster network like this.::

   $ sudo ifconfig eth0 0.0.0.0

Then, stonith-<worker_hostname> resource will pick up the failure of the node
and run ssh fence job through fencing network.


