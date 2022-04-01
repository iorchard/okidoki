Deploy masakari services using helm chart
==============================================

This is a guide to deploy masakari services with custom-built images
using helm chart in TACO environment.

Pre-requisites
---------------

* TACO should be already installed.

Deploy masakari helm chart using armada
------------------------------------------

TACO uses armada as a deployment tool.

Copy maskari helm chart to your taco helm chart location.::

   $ cp -a <path/to/okidoki>/taco/openstack-helm/masakari \
            <path/to/taco>/charts/openstack-helm/

There is a armada manifest file (masakari-manifest.yaml) for masakari 
in okidoki/taco/armada.

Edit the file to change password and location of source.::

   $ vi <path/to/okidoki>/taco/armada/masakari-manifest.yaml
   source:
     type: local
     location: /home/clex/taco/charts/openstack-helm-infra
     subpath: helm-toolkit
   ...
     conf:
      db_acl:
        enabled: true
        allow_network: 10.233.64.0/18
       masakari:
         DEFAULT:
           debug: true
           os_project_domain_name: default
           os_privileged_user_tenant: admin
           os_privileged_user_name: admin
          os_privileged_user_password: <password>
          # shorten the wait time after node failure to 60 seconds
          # at your own risk!!!
          #wait_period_after_service_update: 60
       masakari_monitors:
         DEFAULT:
           debug: true
         auth:
           admin:
             username: admin
             password: <password>
           masakari:
             username: masakari
             password: <password>
         port:
           api:
             default: 8080
         auth:
           admin:
             username: root
             password: <password>
           masakari:
             username: masakari
             password: <password>
       oslo_messaging:
         auth:
           admin:
             username: rabbitmq
             password: <password>
           masakari:
             username: masakari
             password: <password>
         statefulset: null
   source:
     type: local

Change <password> to the right password.


Run armada.::

   $ sudo docker exec -u root armada armada apply \
      --tiller-host <any_controller_ip> \
      --tiller-port 32134 \
      --timeout 600 \
      <path/to/okidoki>/taco/armada/masakari-manifest.yaml

See helm list.::

   $ helm list 'masakari+'
   NAME         	REVISION	UPDATED                 	STATUS  	CHART         	APP VERSION	NAMESPACE
   taco-masakari	1       	Tue Jan  5 13:53:33 2021	DEPLOYED	masakari-0.1.0	           	openstack


Get the pods status.::

   $ kubectl get pods -n openstack -l application=masakari
   NAME                               READY   STATUS      RESTARTS   AGE
   masakari-api-7b8f684d8f-xpgv6      1/1     Running     0          78s
   masakari-db-init-6hhxl             0/1     Completed   0          78s
   masakari-db-sync-qfz22             0/1     Completed   0          78s
   masakari-engine-58699897d4-sk2xv   1/1     Running     0          78s
   masakari-hostmonitor-4jqxd         1/1     Running     0          78s
   masakari-hostmonitor-pvld6         1/1     Running     0          78s
   masakari-instancemonitor-qk4pj     1/1     Running     0          78s
   masakari-instancemonitor-rn8nr     1/1     Running     0          78s
   masakari-ks-endpoints-whz5j        0/3     Completed   0          78s
   masakari-ks-service-ql7tx          0/1     Completed   0          78s
   masakari-ks-user-kmlss             0/1     Completed   0          78s
   masakari-rabbit-init-bmkhc         0/1     Completed   0          78s

Masakari initial setup
------------------------

Add masakari endpoint host in /etc/hosts.::

   $ sudo vi /etc/hosts
   ...
   <controller_vip>  masakari.openstack.svc.cluster.local

Go to openstack client shell.::

   $ tacos
   root@99ea69d1a7b9:/#

Create a segment.::

   # openstack segment create okidoki auto COMPUTE

Create all compute hosts in a segment.::

   # openstack segment host create <compute_hostname> COMPUTE SSH okidoki

Manual process after evacuation from host failure
----------------------------------------------------

When hostmonitor on other nodes detects HA cluster failure of the host, 
it sends a notification to masakari-api and masakari-engine picks up the
notification and process to evacuate VM instances on the failed host.

The masakari-engine 

#. sets on_maintenance flag for the failed host in masakari database and
#. disables compute service of the failed host and
#. wait for 3 minutes for openstack to make the failed host in down state.
#. Then, it evacuates VM instances of the failed host using nova api and
#. confirms VM instances are evacuated well and 
#. finally, it sets the notification state to finished.

After the failed host is booted, it cannot run VM instance since its compute
service is disabled. So do the following manual processes to make the failed
host go back to normal compute service.

#. Set nova-compute service to enable.::

   $ openstack compute service set --enable <hostname> nova-compute

#. Set on_maintenance to False for masakari segment.::

   $ openstack segment host update --on_maintenance False okidoki <hostname>

#. Clean up resource and stonith history.::

   $ sudo pcs resource cleanup
   $ sudo pcs stonith cleanup

#. Confirm the host is in online state for remote node.::

   $ sudo pcs status nodes both
   ...
   Pacemaker Remote Nodes:
     Online: <hostname> <hostname> ...

