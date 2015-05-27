Gap analysis in upstream projects
=================================

This part presents the findings of gaps in upstream projects, e.g. OpenStack.

OpenStack Nova
--------------

Race condition from multiple running scheduler instances
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'scalability issue'
* Description:

  + To-be

    - Eliminate the race condition when running multiple scheduler instances to do HA.

  + As-is:

    - It is needed to use more than one scheduler instances to do HA when the cluster becomes large. If there are multiple Nova Schedulers (such as Load Balance) and they are not synchronized for messaging one may wait for resource whereas other may be locking it and not releasing, causing a race condition.

  + Gap

    - Multiple running scheduler instances can cause a race condition.

Health status of compute node
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'deficiency in performance'
* Description:

  + To-be

    - Provide all the health status of compute nodes.

  + As-is

    - Currently, while performing some actions like evacuation, Nova is checking for the compute service. If the service is down, it is assumed the host is down. This is not exactly true, since there is a possibility to only have compute service down, while all VMs that are running on the host, are actually up.
    - There is no way to distinguish between two really different things: host status and nova-compute status, which is deployed on the host. Also, provided host information by API and commands, are service centric - i.e. "nova host-list" is just another wrapper for "nova service-list" with different format (in fact "service-list" is super set to "host-list").

  + Gap

    - Not all the health information of compute nodes can be provided. Seems like nova is treating *host* term equally to *compute-host*, which might be misleading. Such situation can be error prone for the case where there is a need to perform host evacuation.

* Related BP:

  + https://blueprints.launchpad.net/nova/+spec/pacemaker-servicegroup-driver
  + https://blueprints.launchpad.net/nova/+spec/host-health-monitoring
  + https://blueprints.launchpad.net/nova/+spec/tooz-for-service-groups

Publish the health status of compute node
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'missing'
* Description:

  + To-be

    - Be able to have the health status of compute nodes published.

  + As-is

    - Nova queries the ServiceGroup API to get the node liveness information.

  + Gap

    - Currently ServiceGroup is keeping the health status of compute nodes internal within nova, could have had those status published to NFV MANO plane.

Active/Active HA of Nova-consoleauth
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'scalability issue'
* Description:

  + To-be

    - To make Nova-consoleauth support active/active HA

  + As-is

    - Both client proxies (web and others) leverage a shared service to manage token authentication called nova-consoleauth. This service must be running for either proxy to work. Many proxies of either type can be run against a single nova-consoleauth service in a cluster configuration.However,it does not support active/active HA currently.

  + Gap

    - Nova-consoleauth doesn't support active/active HA

Store consoleauth tokens to the database
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'deficiency in performance'
* Description:

  + To-be

    - Change the consoleauth service to store the tokens in the database and, optionally, cache them in memory as it does now for fast access.

  + As-is

    - Currently the consoleauth service is storing the tokens and the connection data only in memory. This behavior makes impossible to have multiple instances of this service in a cluster as there is no way for one of the isntances to know the tokens issued by the other.
    - The consoleauth service can use a memcached server to store those tokens, but again, if we want to share them among different instances of it we would be relying in one memcached server which makes this solution unsuitable for a highly available architecture where we should be able to replicate all of the services in our cluster.

  + Gap

    - The consoleauth service is storing the tokens and the connection data only in memory. This behavior makes impossible to have multiple instances of this service in a cluster as there is no way for one of the instances to know the tokens issued by the other.

* Related BP

  + https://blueprints.launchpad.net/nova/+spec/consoleauth-tokens-in-db

OpenStack Neutron
-----------------

DVR and L3 HA
^^^^^^^^^^^^^

* Type: 'missing'
* Description:

  + To-be

    - Allow DVR to support L3 HA, implemented as extensions or drivers which based on VRRP.

  + As-is

    - The distributed virtual router feature has been available in Juno.However,DVR is not compatible with L3 HA currently.

  + Gap

    - The DVR is not compatible with L3 HA.

OpenSatck Cinder
----------------

Active/Active HA of cinder-volume
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'scalability issue'
* Description:

  + To-be

    - Cinder-volume can run in an active/active configuration.

  + As-is

    - Only one cinder-volume instance can be active. Failover to be handled by external mechanism such as pacemaker/corosync.

  + Gap

    - Cinder-volume doesn't supprt active/active configuration.

* Related BP

  + https://review.openstack.org/#/c/124205/
  + https://review.openstack.org/#/c/147879/

Cinder volume multi-attachment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'missing'
* Description:

  + To-be

    - Cinder volumes can be attached to multiple VMs at the same time.

  + As-is

    - Cinder volumes can only be attached to one VM at a time.

  + Gap

    - Nova and cinder do not allow for multiple simultaneous attachments.

* Related BP

  + https://blueprints.launchpad.net/openstack/?searchtext=multi-attach-volume

VIM Northbound Interface
------------------------

NFVI level correlation of persistent VNF failures
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Type: 'missing'
* Description:

  + To-be

    - Provide a VIM API to report errors that cannot be recovered at the VNF level.
    - VIM correlates reported VNF errors to detect NFVI faults propagated to the VNF.
  + As-is

    - No error report API exists and no fault correlation is performed for failures detected at the VNF level.

  + Gap

    - No error report API exists and no fault correlation is performed for failures detected at the VNF level.

Others
------

QoS management
^^^^^^^^^^^^^^

* Type: 'scalability issue'
* Description:

  + To-be

    - When considering QoS in OPNFV/OpenStack, we should look beyond networks and at all of the resources on which there is contention.
    - It is needed to establish an integrated centralized QoS management system or module.
    - QoS management may include QoS spec defining, QoS monitoring, quality assurance, etc.

  + As-is

    - A QoS spec framework has been provided in Cinder.
    - And Quotas are used in nova to spacify the QoS settings of a VM. However a quota is a maximum amount set of a resource that a user is allowed to use. This does not necessarily mean that the user is guaranteed that much of the given resource, it just means that is the most they can have.  Quotas can sometimes be manipulated to provide a type of QoS.
    - There are a number of Neutron plugins that have their own quality of service API extension, but each of them has their own parameters and structure.

  + Gap

    - Not all components in OpenStack have QoS feature. It is needed to establish an integrated centralized QoS management system or module.

* Related BP

  + https://blueprints.launchpad.net/neutron/+spec/quantum-qos-api
  + https://blueprints.launchpad.net/neutron/+spec/ml2-qos
  + https://blueprints.launchpad.net/nova/+spec/flavor-quota-memory

**Documentation tracking**

Revision: _sha1

Build date:  _date

