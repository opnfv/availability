
This secssion about VIM High availability

============================
5     VIM High availability
============================
VIM in NFV reference architecture  contains all control nodes of openstack, SDN controllers and hardware controllers. It connect  NFVI to VNFM and NFVO. To guarantee high availabilty of VIM is basic requirement on OPNFV platform. Also VIM should provide some mechanism for VNFs used to achieve their high availability.


5.1 Architecture requirement of VIM HA
---------------------------------------
In architecture level all control nodes should avoid single point failure and management network plane which connect control nodes should also be redundant. Control nodes which are stateless like nova-API, glance-API etc. may be redundant simply. But stateful nodes like MySQL, Rabbit MQ, SDN controller may have complex redundant policy.  Different scale of cloud may also require different HA policy. 

Requirement:
------------
- Active-standby redundancy policy would be acceptable in small scale scenario. 

- In large scale scenario all stateful nodes like database nodes, message queue nodes ,SDN controller should be deployed in cluster mode which support all active, N+1 or N+M redundancy.

- In large scale scenario all stateless nodes like nova-api, glance-api etc. should be deployed in all active mode. Loadbalance nodes which introduced for all active mode should also avoid single point failure.


- All control node servers shall have at least two network ports to connect to different networks plane.These ports shall work in binding manner.

- Controllers must be deployed in redundant pairs and have fast fail over times.  Sub 5 seconds.  Monitoring of key software process is required.


5.2 Fault detection and alarm requirement of VIM
--------------------------------------------------
Redundant architecture may provides function continuity for the VIM. For maintenance considerations all failures in the VIM should be detected and alarms should be triggered to NFVO, VNFM and OSS. 

Requirement:
------------
- All hardware failures of control nodes should be detected and alarms should be trigered. OSS, NFVO and VNFM can subscribe these alarms.

- Software on control nodes like Openstack or ODL should be monitored by the clustering software and alarms should be trigered  when exceptions are detected.

- Software on compute nodes like openstack/nova agents, ovs should be monitored by watchdog. When exceptions are detected the software should be restored automatically and alarms should be trigered .

- All alarm indicators should include: Failure time, Failure location, Failure type, Failure level. .

- Consumer can subscribe to alarms through some mechanism in VIM online.

- All alarms should be kept persistently for future inquiry.  
- VIM should distinguish between the failure of the compute host and the failure of the host hw[gap 2]
- VIM should be able to publish the health status of the compute node to NFV MANO[gap 3]

5.3 HA mechanism of VIM provided for VNFs NFV
------------------------------------------------
When VNFs approach their HA scheme, they usually require underlayer resource to provide some mechanism for helping just like hardware watchdog helps to HA of traditional network devices. Also virtualization introduces some other requirement like affinity and anti-affinity when virtual resource is allocated.

Requirement
------------
- Resources with specific HA functions like watchdog enable, redundant network port or any more should be properly tagged exposed to VNFM

- VIM should provide anti-affinity scheme for VNF to deploy redundant service on diffirent computing node or availability zone.

- VIM should be able to deploy classified virtual resources to VNFs following the SAL discribtion in VNFD

- there should be limitation on the collocation of VMs on a single server to reduce failure impact zone.

- VIM should provide data collection to calculate the HA related metrics for VNFs

- VIM should support the VNF/VNFM to initiate the repair/reboot of resources of the NFVI

- VIM should correlate the failures detected on collocated virtual resources to identify latent faults in HW and virtualization facilities 
- It should be possible to disallow the live migration of VMs and when it is allowed it should be possible to specify the tolerated interruption time.
- It should be possible to restrict the simultaneous migration of VMs hosting a given VNF
- VNFM/VNF can trigger the VIM to scale in/out
- Scheduler in VIM should require Active/active HA scheme or use load balancing. Race - condition from multiple running scheduler instances should be eliminate [GAP 1]
- VIM should be able to trigger the evacuation of the VM before bringing the host down when maintenance mode is set for the compute host.[gap 11]
- consoleauth should support active/active HA, and the token should be stored in the database[GAP  4, Gap 5] 
- VIM should be able to restore a new VM when the original VM fail.
- VIM should support policies to prioritize a certain VNF.
- VIM should be able to provide classified virtual resources to VNFs in different SAL.

5.4 SDN controller
-------------------
SDN controller: Distributed or Centralized

Requriements
-------------
- SDN controller must be deployed as redundant pair in the centralized model.
- In a distributed model, mastership election must look after which node is in overall control.
- For distributed model, APP should not be aware of HA of controller. That is it is a - logically centralized system for NBI.
- Also event notification is needed

5.5 QoS management
-------------------
There are a lot of services working together to provide NFVI and VIM functionalities. As a service, the quality of a service should be considered. "The term quality of service traditionally refers to the users reservation, or guarantee of a certain amount of network bandwidth." When considering QoS in OPNFV we really should look beyond networks and at all of the resources on which there is contention.

Requirement:
------------
- According to the policy settings, QoS specifications (API) should be defined first.
- Once a QoS spec is defined, the monitoring capability is required to make sure that whether the QoS requirement is met. 
- The monitoring capabilities required to support operations in this area include the ability to measure QoS parameters, storing and analysing data, resource consumption measuring and assessment.
- To achieve quality assurance, some other mechanisms can be used, e.g. quotas, resource reservation, resource limitation, etc.
 