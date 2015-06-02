
This section about VIM High availability

============================
5     VIM High availability
============================
VIM in NFV reference architecture  contains all control nodes of OpenStack, SDN controllers and
hardware controllers. It manages the NFVI according to the instructions/requests of the VNFM and
NFVO and reports them back about the NFVI status. To guarantee the high availability of VIM is
a basic requirement on OPNFV platform. Also VIM should provide some mechanism for VNFs to achieve
their own high availability.


5.1 Architecture requirement of VIM HA
---------------------------------------
The architecture of the control nodes should avoid single point of failure and management
network plane which connects the control nodes should also be redundant. Services in control node
which are stateless like nova-API, glance-API etc. should be redundant but without data synchronization.
Stateful services like MySQL, Rabbit MQ, SDN controller may have complex redundant policy.
Different scale of cloud may also require different HA policy.

Requirement:
------------
- In small scale scenario active-standby redundancy policy would be acceptable.

- In large scale scenario all stateful services like database, message queue, SDN controller
  should be deployed in cluster mode which support all active, N+M active-standby redundancy.

- In large scale scenario all stateless services like nova-api, glance-api etc. should be deployed
  in all active mode. Load balance nodes which introduced for all active mode should also avoid
  single point of failure.

- All control node servers shall have at least two network ports to connect to different networks
  plane. These ports shall work in bonding manner.

- Services in redundant pairs can switch over in less than 5 seconds.

- Status of services must be monitored.


5.2 Fault detection and alarm requirement of VIM
--------------------------------------------------
Redundant architecture may provide function continuity for the VIM. For maintenance considerations
all failures in the VIM should be detected and notifications should be triggered to NFVO, VNFM and other
VIM consumers.

Requirement:
------------
- All hardware failures of control nodes should be detected and relevant alarms should be triggered. 
  OSS, NFVO and VNFM can subscribe these alarms.

- Software on control nodes like OpenStack or ODL should be monitored by the clustering software
  and alarms should be triggered when exceptions are detected.

- Software on compute nodes like OpenStack/nova agents, ovs should be monitored by watchdog. When
  exceptions are detected the software should be restored automatically and alarms should be triggered.

- All alarm indicators should include: Failure time, Failure location, Failure type, Failure level.

- The VIM should provide an interface through which consumers can subscribe to alarms and notifications.

- All alarms should be kept for future inquiry in VIM, ageing policy of these records should be configurable.

- VIM should distinguish between the failure of the compute node and the failure of the host HW.

- VIM should be able to publish the health status of the compute node to NFV MANO.

5.3 HA mechanism of VIM provided for VNFs
------------------------------------------------
When VNFs deploy their HA scheme, they usually require from underlying resource to provide some mechanism.
This is similar to the hardware watchdog in the traditional network devices. Also virtualization
introduces some other requirement like affinity and anti-affinity when virtual resource is allocated.

Requirement
------------
- Resources with specific HA functions like watchdog enable, redundant network ports and etc. should
  be properly tagged and exposed to VNF and VNFM with standard API.

- VIM should provide anti-affinity scheme for VNF to deploy redundant service on diffirent level of 
  aggregation of resource.

- VIM should be able to deploy classified virtual resources to VNFs following the SAL description in VNFD.

- VIM should provide data collection to calculate the HA related metrics for VNFs.

- VIM should support the VNF/VNFM to initiate the operation of resources of the NFVI, such as repair/reboot.

- VIM should correlate the failures detected on collocated virtual resources to identify latent faults in 
  HW and virtualization facilities

- VIM should be possible to disallow the live migration of VMs and when it is allowed it should be possible
  to specify the tolerated interruption time.

- VIM should be possible to restrict the simultaneous migration of VMs hosting a given VNF.

- VIM should provide the APIs to trigger scale in/out to VNFM/VNF.

- Scheduler in VIM should require Active/active HA scheme or use load balancing. Race - condition from
  multiple running scheduler instances should be eliminate

- VIM should be able to trigger the evacuation of the VMs before bringing the host down when maintenance mode
  is set for the compute host.

- VIM should configure Consoleauth in active/active HA mode, and should store the token in database.

- VIM should be able to restore a new VM when the original VM fail and the new VM should be kept in the same 
  initial work state as the failed VM.

- VIM should support policies to prioritize a certain VNF.

5.4 SDN controller
-------------------
SDN controller: Distributed or Centralized

Requriements
-------------
- In centralized model SDN controller must be deployed as redundant pairs.

- In distributed model, mastership election must look after which node is in overall control.

- For distributed model, APP should not be aware of HA of controller. That is it is a - logically centralized
  system for NBI.

- Event notification is required as section 5.2 mentioned.

