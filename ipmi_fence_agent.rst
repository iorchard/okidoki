IPMI fence agent
=========================

Pre-requisites
---------------

HA cluster should be set up and running.

There should be at least two networks - HA cluster network and fencing network

* HA Cluster network: corosync membership network
* Fencing network: the network to use for fencing (ssh, IPMI, etc.)

Install
----------

Install the fence-agents-ipmilan pacakge in all controller nodes.::

   $ sudo yum install -y fence-agents-ipmilan

Set ipmi user password env variable.::

   $ read -s -p 'ipmi user pw: ' USERPW
   $ export USERPW

Create fence devices for all worker nodes in pacemaker.::

   $ sudo -E pcs stonith create stonith-<worker_hostname> \
      fence_ipmilan pcmk_host_list="<worker_hostname>" \
      ipaddr="<worker_ipmi_ip>" \
      login="<ipmi_username>" passwd="$USERPW" lanplus=true

   $ sudo pcs constraint location add stonith-<worker_hostname>-loc \
      stonith-<worker_hostname> <worker_hostname> INFINITY


USERPW should be the <ipmi_username>'s password on each worker node.

Replace worker_hostname and username appropriately.

<worker_ipmi_ip> should be the ip address of fencing network.

Enable stonith and set stonith-action=off to power off the node instead
of rebooting.::

   $ sudo pcs property set stonith-enabled=true
   $ sudo pcs property set stonith-action=off

See the cluster status. It shows stonith resources.

Verify the stonith configuration.::

   $ sudo pcs stonith

Test fencing
--------------

There are many ways to test fencing.

* Cut the HA cluster network connection of one of worker nodes.

Let's say worker1's HA cluster network interface is eth0.
Cut the HA cluster network like this.::

   $ sudo ifconfig eth0 0.0.0.0

* Trigger kernel panic.

Force to trigger kernel panic on one of worker ndoes.::

   $ echo c | sudo tee /proc/sysrq-trigger

Then, stonith-<worker_hostname> resource will pick up the failure of the node
and run fence job through fencing network.

