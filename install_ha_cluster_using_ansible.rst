Install HA cluster using ansible
=================================

This is a guide to install HA cluster using ansible playbook.


Pre-requisites
------------------

Edit hosts.ini in /path/to/okidoki/taco2/ansible.

* okidoki_controllers: TACO2 controller nodes
* okidoki_workers: TACO2 compute nodes

Run
----
Run okidoki playbook.::

   $ cd /path/to/okidoki/taco2/ansible
   $ ansible-playbook -i hosts.ini site.yml


