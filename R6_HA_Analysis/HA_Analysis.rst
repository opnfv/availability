.. image:: opnfv-logo.png
  :height: 40
  :width: 200
  :alt: OPNFV
  :align: left

============
High Availability Requirement Analysis in OPNFV
============

******************
1 Introduction
******************
This High Availability Requirement Analysis Document is used for eliciting High Availability
Requirements of OPNFV. The document will refine high-level High Availability goals, into
detailed HA mechanism design. And HA mechanisms are related with potential failures on
different layers in OPNFV. Moreover, this document can be used as reference for HA Testing
scenarios design.
A requirement engineering model KAOS is used in this document.

******************
2 Terminologies and Symbols
******************
The following concepts in KAOS will be used in the diagrams of this document.

- **Goal**: The objective to be met by the target system.

- **Obstacle**: Condition whose satisfaction may prevent some goals from being achieved.

- **Agent**: Active Object performing operations to achieve goals.

- **Requirement**: Goal assigned to an agent of the software being studied.

- **Domain Property**: Descriptive assertion about objects in the environment of the software.

- **Refinement**: Relationship linking a goal to other goals that are called its subgoals.
  Each subgoal contributes to the satisfaction of the goal it refines. There are two types of
  refinements: AND refinement and OR refinement, which means whether the goal can be archived by
  satisfying all of its sub goals or any one of its sub goals.

- **Conflict**: Relationship linking an obstacle to a goal if the obstacle obstructs the goal
  from being satisfied.

- **Resolution**: Relationship linking a goal to an obstacle if the goal can resolve the
  obstacle.

- **Responsibility**: Relationship between an agent and a requirement. Holds when an agent is
  assigned the responsibility of achieving the linked requirement.

Figure 1 shows how these concepts are displayed in a KAOS diagram.

.. figure:: images/KAOS_Sample.png
    :alt: KAOS Sample
    :figclass: align-center

    Fig 1. A KAOS Sample Diagram

******************
3 High Availability Goals of OPNFV
******************

3.1 Overall Goals
>>>>>>>>>>>>>>>>>>

The Final Goal of OPNFV High Availability is to provide high available VNF services. And the
following objectives are required to meet:

- There should be no single point of failure in the NFV framework.

- All resiliency mechanisms shall be designed for a multi-vendor environment, where for example
  the NFVI, NFV-MANO, and VNFs may be supplied by different vendors.

- Resiliency related information shall always be explicitly specified and communicated using
  the reference interfaces (including policies/templates) of the NFV framework.



3.2 Service Level Agreements of OPNFV HA
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

Service Level Agreements of OPNFV HA are mainly focused on time constraints of service outage,
failure detection, failure recovery. The following table outlines the SLA metrics of different
service availability levels described in ETSI GS NFV-REL 001 V1.1.1 (2015-01). Table 1 shows
time constraints of different Service Availability Levels. In this document, SAL1 is the
default benchmark value required to meet.

*Table 1. Time Constraints for Different Service Availability Levels*

+--------------------------------+----------------------------+------------------------+
| Service Availability Level     | Failure Detection Time     | Failure Recovery Time  |
+================================+============================+========================+
| SAL1                           | <1s                        | 5-6s                   |
+--------------------------------+----------------------------+------------------------+
| SAL2                           | <5s                        | 10-15s                 |
+--------------------------------+----------------------------+------------------------+
| SAL3                           | <10s                       | 20-25s                 |
+--------------------------------+----------------------------+------------------------+


******************
4 Overall Analysis
******************
Figure 2 shows the overall decomposition of high availability goals. The high availability of
VNF Services can be refined to high availability of VNFs, MANO, and the NFVI where VNFs are
deployed; the high availability of NFVI Service can be refined to high availability of Virtual
Compute Instances, Virtual Storage and Virtual Network Services; the high availability of
virtual instance is either the high availability of containers or the high availability of VMs,
and these high availability goals can be further decomposed by how the NFV environment is
deployed.

.. figure:: images/Total_Framework.png
    :alt: Overall HA Analysis of OPNFV
    :figclass: align-center

    Fig 2. Overall HA Analysis of OPNFV

Thus the high availability requirement of VNF services can be classified into high availability
requirements on different layers in OPNFV. The following layers are mainly discussed in this
document:

- VNF HA

- MANO HA

- Virtual Infrastructure HA (container HA or VM HA)

- VIM HA

- SDN HA

- Hypervisor HA

- Host OS HA

- Hardware HA

The next section will illustrate detailed analysis of HA requirements on these layers.

******************
5 Detailed Analysis
******************

5.1 VNF HA
>>>>>>>>>>>>>>>>>>

.. TBD

5.2 MANO HA
>>>>>>>>>>>>>>>>>>

.. TBD

5.3 Virtual Infrastructure HA
>>>>>>>>>>>>>>>>>>


The Virtual Infrastructure HA in OPNFV includes container HA and VM HA.

Container HA
::::::::::::::::::::::::::::

The container HA in OPNFV is mainly focus on Kubernetes(K8s) platform. And using the Pod as
the smallest unit of management, creation, and planning, the K8s' container HA actually means
the High Availability of running Pods.

Table 2 shows the potential faults of running pods in K8s. when it happens, the ReplicationController
or ReplicaSet can prevent the services provided by the pod from being unavailable, as is shown in
figure 3.

*Table 2. Potential Faults in VIM level*

+------------+--------------+----------------------------------------------------+----------------+
| Service    | Fault        | Description                                        | Severity       |
+============+==============+====================================================+================+
|            |              | All Containers in the Pod have terminated, and     |                |
| Running by | Pod failure  | at least one Container has terminated in failure.  | Critical       |
| pods       |              | That is, the Container either exited with non-zero |                |
|            |              | status or was terminated by the system.            |                |
+------------+--------------+----------------------------------------------------+----------------+

.. figure:: images/Container_HA_analysis_in_K8s.png
    :alt: VIM HA Analysis
    :figclass: align-center

    Fig 3. Container HA analysis in K8s
    
    
The Replication Controller or ReplicaSet (ReplicaSet is the next-generation Replication Controller) 
is a kind of K8s Master Components, which ensures that a specified number of pod replicas are running 
at any one time.

The following requirements are elicited for Pod HA:

**[Req 5.3.1]** A pod or a homogeneous set of pods is always up and available until terminated properly.

**[Req 5.3.2]** The ReplicationController or ReplicaSet should terminate the extra pods If there are 
more pods than specified number.

**[Req 5.3.3]** The ReplicationController or ReplicaSet should start more pods If there are fewer pods 
than specified number. 

**[Req 5.3.4]** The new Pod should be scheduled to other Nodes, if detecting the failure state of the 
host or container.


5.4 VIM HA
>>>>>>>>>>>>>>>>>>

The VIM in the NFV reference architecture contains different components of Openstack, SDN
controllers and other virtual resource controllers. VIM components can be classified into three
types:

- **Entry Point Components**: Components that give VIM service interfaces to users, like nova-
  api, neutron-server.

- **Middlewares**: Components that provide load balancer services, messaging queues, cluster
  management services, etc.

- **Subcomponents**: Components that implement VIM functions, which are called by Entry Point
  Components but not by users directly.

Table 2 shows the potential faults that may happen on VIM layer. Currently the main focus of
VIM HA is the service crash of VIM components, which may occur on all types of VIM components.
To prevent VIM services from being unavailable, Active/Active Redundancy, Active/Passive
Redundancy and Message Queue are used for different types of VIM components, as is shown in
figure 3.

*Table 2. Potential Faults in VIM level*

+------------+------------------+-------------------------------------------------+----------------+
| Service    | Fault            | Description                                     | Severity       |
+============+==================+=================================================+================+
| General    | Service Crash    | The processes of a service crashed unnormally.  | Critical       |
+------------+------------------+-------------------------------------------------+----------------+

.. figure:: images/VIM_Analysis.png
    :alt: VIM HA Analysis
    :figclass: align-center

    Fig 3. VIM HA Analysis


Active/Active Redundancy
::::::::::::::::::::::::::::
Active/Active Redundancy manages both the main and redundant systems concurrently. If there is
a failure happens on a component, the backups are already online and users are unlikely to
notice that the failed VIM component is under fixing. A typical Active/Active Redundancy will
have redundant instances, and these instances are load balanced via a virtual IP address and a
load balancer such as HAProxy.

When one of the redundant VIM component fails, the load balancer should be aware of the
instance failure, and then isolate the failed instance from being called until it is recovered.
The requirement decomposition of Active/Active Redundancy is shown in Figure 4.

.. figure:: images/Active_Active_Redundancy.png
    :alt: Active/Active Redundancy Requirement Decomposition
    :figclass: align-center

    Fig 4. Active/Active Redundancy Requirement Decomposition

The following requirements are elicited for VIM Active/Active Redundancy:

**[Req 5.4.1]** Redundant VIM components should be load balanced by a load balancer.

**[Req 5.4.2]** The load balancer should check the health status of VIM component instances.

**[Req 5.4.3]** The load balancer should isolate the failed VIM component instance until it is
recovered.

**[Req 5.4.4]** The alarm information of VIM component failure should be reported.

**[Req 5.4.5]** Failed VIM component instances should be recovered by a cluster manager.

Table 3 shows the current VIM components using Active/Active Redundancy and the corresponding
HA test cases to verify them.

*Table 3. VIM Components using Active/Active Redundancy*

+-------------------+-------------------------------------------------------+----------------------+
| Component         | Description                                           | Related HA Test Case |
+===================+=======================================================+======================+
| nova-api          | endpoint component of Openstack Compute Service Nova  | yardstick_tc019      |
+-------------------+-------------------------------------------------------+----------------------+
| nova-novncproxy   | server daemon that serves the Nova noVNC Websocket    |                      |
|                   | Proxy service, which provides a websocket proxy that  |                      |
|                   | is compatible with OpenStack Nova noVNC consoles.     |                      |
+-------------------+-------------------------------------------------------+----------------------+
| neeutron-server   | endpoint component of Openstack Networking Service    | yardstick_tc045      |
|                   | Neutron                                               |                      |
+-------------------+-------------------------------------------------------+----------------------+
| keystone          | component of Openstack Identity Service Service       | yardstick_tc046      |
|                   | Keystone                                              |                      |
+-------------------+-------------------------------------------------------+----------------------+
| glance-api        | endpoint component of Openstack Image Service Glance  | yardstick_tc047      |
+-------------------+-------------------------------------------------------+----------------------+
| glance-registry   | server daemon that serves image metadata through a    |                      |
|                   | REST-like API.                                        |                      |
+-------------------+-------------------------------------------------------+----------------------+
| cinder-api        | endpoint component of Openstack Block Storage Service | yardstick_tc048      |
|                   | Service Cinder                                        |                      |
+-------------------+-------------------------------------------------------+----------------------+
| swift-proxy       | endpoint component of Openstack Object Storage        | yardstick_tc049      |
|                   | Swift                                                 |                      |
+-------------------+-------------------------------------------------------+----------------------+
| horizon           | component of Openstack Dashboard Service Horizon      |                      |
+-------------------+-------------------------------------------------------+----------------------+
| heat-api          | endpoint component of Openstack Stack Service Heat    |                      |
+-------------------+-------------------------------------------------------+----------------------+
| mysqld            | database service of VIM components                    |                      |
+-------------------+-------------------------------------------------------+----------------------+

Active/Passive Redundancy
::::::::::::::::::::::::::::

Active/Passive Redundancy maintains a redundant instance that can be brought online when the
active service fails. A typical Active/Passive Redundancy maintains replacement resources that
can be brought online when required. Requests are handled using a virtual IP address (VIP) that
facilitates returning to service with minimal reconfiguration. A cluster manager (such as
Pacemaker or Corosync) monitors these components, bringing the backup online as necessary.

When the main instance of a VIM component is failed, the cluster manager should be aware of the
failure and switch the backup instance online. And the failed instance should also be recovered
to another backup instance. The requirement decomposition of Active/Passive Redundancy is shown
in Figure 5.

.. figure:: images/Active_Passive_Redundancy.png
    :alt: Active/Passive Redundancy Requirement Decomposition
    :figclass: align-center

    Fig 5. Active/Passive Redundancy Requirement Decomposition

The following requirements are elicited for VIM Active/Passive Redundancy:

**[Req 5.4.6]** The cluster manager should replace the failed main VIM component instance with
a backup instance.

**[Req 5.4.7]** The cluster manager should check the health status of VIM component instances.

**[Req 5.4.8]** Failed VIM component instances should be recovered by the cluster manager.

**[Req 5.4.9]** The alarm information of VIM component failure should be reported.


Table 4 shows the current VIM components using Active/Passive Redundancy and the corresponding
HA test cases to verify them.

*Table 4. VIM Components using Active/Passive Redundancy*

+-------------------+-------------------------------------------------------+----------------------+
| Component         | Description                                           | Related HA Test Case |
+===================+=======================================================+======================+
| haproxy           | load balancer component of VIM components             | yardstick_tc053      |
+-------------------+-------------------------------------------------------+----------------------+
| rabbitmq-server   | messaging queue service of VIM components             | yardstick_tc056      |
+-------------------+-------------------------------------------------------+----------------------+
| corosync          | cluster management component of VIM components        | yardstick_tc057      |
+-------------------+-------------------------------------------------------+----------------------+

Message Queue
::::::::::::::::::::::::::::
Message Queue provides an asynchronous communication protocol. In Openstack, some projects (
like Nova, Cinder) use Message Queue to call their sub components. Although Message Queue
itself is not an HA mechanism, how it works ensures the high availability when redundant
components subscribe to the Message Queue. When a VIM sub component fails, since there are
other redundant components are subscribing to the Message Queue, requests still can be processed.
And fault isolation can also be archived since failed components won't fetch requests actively.
Also, the recovery of failed components is required. Figure 6 shows the requirement
decomposition of Message Queue.

.. figure:: images/Message_Queue.png
    :alt: Message Queue Requirement Decomposition
    :figclass: align-center

    Fig 6. Message Queue Redundancy Requirement Decomposition

The following requirements are elicited for Message Queue:

**[Req 5.4.10]** Redundant component instances should subscribe to the Message Queue, which is
implemented by the installer.

**[Req 5.4.11]** Failed VIM component instances should be recovered by the cluster manager.

**[Req 5.4.12]** The alarm information of VIM component failure should be reported.

Table 5 shows the current VIM components using Message Queue and the corresponding HA test cases
to verify them.

*Table 5. VIM Components using Messaging Queue*

+-------------------+-------------------------------------------------------+----------------------+
| Component         | Description                                           | Related HA Test Case |
+===================+=======================================================+======================+
| nova-scheduler    | Openstack compute component determines how to         |                      |
|                   | dispatch compute requests                             |                      |
+-------------------+-------------------------------------------------------+----------------------+
| nova-cert         | Openstack compute component that serves the Nova Cert |                      |
|                   | service for X509 certificates. Used to generate       |                      |
|                   | certificates for euca-bundle-image.                   |                      |
+-------------------+-------------------------------------------------------+----------------------+
| nova-conductor    | server daemon that serves the Nova Conductor service, |                      |
|                   | which provides coordination and database query        |                      |
|                   | support for Nova.                                     |                      |
+-------------------+-------------------------------------------------------+----------------------+
| nova-compute      | Handles all processes relating to instances (guest    |                      |
|                   | vms). nova-compute is responsible for building a disk |                      |
|                   | image, launching it via the underlying virtualization |                      |
|                   | driver, responding to calls to check its state,       |                      |
|                   | attaching persistent storage, and terminating it.     |                      |
+-------------------+-------------------------------------------------------+----------------------+
| nova-consoleauth  | Openstack compute component for Authentication of     |                      |
|                   | nova consoles.                                        |                      |
+-------------------+-------------------------------------------------------+----------------------+
| cinder-scheduler  | Openstack volume storage component decides on         |                      |
|                   | placement for newly created volumes and forwards the  |                      |
|                   | request to cinder-volume.                             |                      |
+-------------------+-------------------------------------------------------+----------------------+
| cinder-volume     | Openstack volume storage component receives volume    |                      |
|                   | management requests from cinder-api and               |                      |
|                   | cinder-scheduler, and routes them to storage backends |                      |
|                   | using vendor-supplied drivers.                        |                      |
+-------------------+-------------------------------------------------------+----------------------+
| heat-engine       | Openstack Heat project server with an internal RPC    |                      |
|                   | api called by the heat-api server.                    |                      |
+-------------------+-------------------------------------------------------+----------------------+


5.5 Hypervisor HA
>>>>>>>>>>>>>>>>>>

.. TBD

5.6 Host OS HA
>>>>>>>>>>>>>>>>>>

.. TBD

5.7 Hardware HA
>>>>>>>>>>>>>>>>>>

.. TBD


******************
6 References
******************

- A KAOS Tutorial: http://www.objectiver.com/fileadmin/download/documents/KaosTutorial.pdf

- ETSI GS NFV-REL 001 V1.1.1(2015-01):
  http://www.etsi.org/deliver/etsi_gs/NFV-REL/001_099/001/01.01.01_60/gs_NFV-REL001v010101p.pdf

- Openstack High Availability Guide: https://docs.openstack.org/ha-guide/

- Highly Available (Mirrored) Queues: https://www.rabbitmq.com/ha.html
