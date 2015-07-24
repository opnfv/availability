==============
X HA Use Cases
==============


With respect to service high availability we need to consider whether a VNF implementation is
statefull or stateless and if it includes an HA manager which handles redundancy or not.
For statefull VNFs we can also distinguish the cases when the state is maintained inside
of the VNF or it is stored in an external shared storage.

Whenever there is an HA management it usually implies a fault detection mechanism, which
triggers the actions necessary for fault isolation followed by the recovery of the
service and the repair of the failed entity in the system. In some cases the recovery
of the service and the repair actions are the same, such as restarting a failed
component. In other cases the repair cannot be performed by the HA manager detecting
the failure. E.g. the VNF cannot repair a failed VM, in  fact due to the layered architecture
the VNF cannot even know whether the VM failed, its hosting hypervisor, or the physical host.
Accordingly a failure may be detected by multiple managers of the different layers of the
system, each of which may want to react to the event. To resolve the problem in a consistent
manner and completely recover from the failure the managers need to collaborate and coordinate
their actions.

Considering all these issues the following basic use cases can be identified (see table 1.).

Table 1: VNF high availability use cases

+---------+-------------------+----------------+-------------------+----------+
|         | VNF Statefullness | VNF Redundancy | Failure detection | Use Case |
+=========+===================+================+===================+==========+
| VNF     | yes               | yes            | VNF level only    | UC1      |
|         |                   |                +-------------------+----------+
|         |                   |                | VNF & NFVI levels | UC2      |
|         |                   +----------------+-------------------+----------+
|         |                   | no             | VNF level only    | UC3      |
|         |                   |                +-------------------+----------+
|         |                   |                | VNF & NFVI levels | UC4      |
|         +-------------------+----------------+-------------------+----------+
|         | no                | yes            | VNF level only    | UC5      |
|         |                   |                +-------------------+----------+
|         |                   |                | VNF & NFVI levels | UC6      |
|         |                   +----------------+-------------------+----------+
|         |                   | no             | VNF level only    | UC7      |
|         |                   |                +-------------------+----------+
|         |                   |                | VNF & NFVI levels | UC8      |
+---------+-------------------+----------------+-------------------+----------+

These use cases assume that the failure is detected in the faulty entity (VNF component
or the VM). However this is not always the case, there is no guarantee that a fault manifests
within the faulty entity. For example a memory leak in one process may impact or even crash
any other process running in the same execution environment. Accordingly, repair a failing
entity (i.e. the crashed process) may not resolve the problem and soon the same or another
entity may fail within this execution environment indicating that the fault remains in the
system. Thus, there is a need of extrapolating the failure to a wider scope and perform the
recovery at that level to get rid of the problem at least temporarily. This requires the
correlation of repeated failures in a wider scope and escalate the recovery action to this
wider scope. In the layered architecture this means that the manager detecting the failure
may not be the one in charge of scope at which it can be resolved, so the escalation needs to
be forwarded to the manager in charge, which brings us to an additional use case UC9.

We need to consider for each of these use cases the events detected, their impact,
and the actions triggered to recover the service provided by the VNF, and to repair the
faulty entity.

We are going to describe each of the listed use cases from this perspective to better
understand how the problem of service high availability can be tackled the best.

Before getting into the details it is worth mentioning the end-to-end service recovery
times recommended by the ETSI NFV REL document [REL] (see table 2.).

*Table 2: Service availability levels (SAL)*

+----+--------------+----------------------+------------------------------------+
|SAL |Service       |Custumer Type         | Recommendation                     |
|    |Recovery      |                      |                                    |
|    |Time          |                      |                                    |
|    |Threshold     |                      |                                    |
+====+==============+======================+====================================+
|1   |5–6 seconds   |Network Operator      |Redundant resources to be           |
|    |              |Control Traffic       |made available on-site to           |
|    |              |                      |ensure fastrecovery.                |
|    |              |Government/Regulatory |                                    |
|    |              |Emergency Services    |                                    |
+----+--------------+----------------------+------------------------------------+
|2   |10–15 seconds |Enterprise and/or     |Redundant resources to be available |
|    |              |large scale customers |as a mix of on-site and off-site    |
|    |              |                      |as appropriate: On-site resources to|
|    |              |Network Operators     |be utilized for recovery of         |
|    |              |service traffic       |real-time service; Off-site         |
|    |              |                      |resources to be utilized for        |
|    |              |                      |recovery of data services           |
+----+--------------+----------------------+------------------------------------+
|3   |20–25 seconds |General Consumer      |Redundant resources to be mostly    |
|    |              |Public and ISP        |available off-site. Real-time       |
|    |              |Traffic               |services should be recovered before |
|    |              |                      |data services                       |
+----+--------------+----------------------+------------------------------------+

Note that even though SAL 1 of [REL] allows for 5-6 seconds of service recovery,
for many services this is too long and such outage causes a service level reset or
the loss of significant amount of data. Therefore at the VNF level the desired
service recovery time is sub-second.


.. [REL] ETSI GS NFV-REL 001 V1.1.1 (2015-01)

***************
X.1 Use Case 1
***************

Use case 1 represents a statefull VNF with redundancy managed by an HA manager,
which is part of the VNF. The fault is detected and handled by this HA manager.

.. figure:: images/Slide4.png
    :alt: VNFC failure in a statefull VNF
    :figclass: align-center

    Fig 1. VNFC failure in a statefull VNF with buit-in HA manager


.. figure:: images/StatefullVNF-VNFCfailure.png
    :alt: MSC of the VNFC failure in a statefull VNF
    :figclass: align-center

    Fig 2. Sequence of events for the use case

