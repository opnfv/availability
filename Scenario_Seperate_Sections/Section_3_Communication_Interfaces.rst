3. Communication Interfaces for VNF HA schemes
===========================================================

This section will discuss some general issues about communication interfaces
in the VNF HA schemes. In sections 2, the usecases of both stateful and
stateless VNFs are discussed. While in this section, we would like to discuss
some specific issues which are quite general for all the usecases proposed
in the previous sections.

3.1. VNF External Interfaces

Regardless whether the VNF is stateful or stateless, all the VNFCs should act as
a union from the perspective of the outside world. That means all the VNFCs should
share a common interface where the outside modules (e.g., the other VNFs) can
access the service from. There could be multiple solutions for this share of IP
interface. However, all of this sharing and switching of IP address should be
ignorant to the outside modules.

There are several approaches for the VNFs to share the interfaces. A few of them
are listed as follows and will be discussed in detail. 

1) IP address of VMs for active/stand-by VM.

2) Load balancers for active/active use cases

Note that combinition of these two approaches is also feasible.

For active/standby VNFCs, there is a common IP address shared by the VMs hosting
the active and standby VNFCs, so that they look as one instance from outside.
The HA manager will manage the assignment of the IP address to the VMs.
(The HA manager may not be aware of this, I.e. the address may be configured
and the active/standby state management is linked to the possession of the IP
address, i.e. the active VNFC claims it as part of becoming active.) Only the
active one possesses the IP address. And when failover happens, the standby
is set to be active and can take possession of the IP address to continue traffic
process.


For active/active VNFCs, a LB(Load Balancer) could be used. In such scenario, there
could be two cases for the deployment and usage of LB.

Case 1: LB used in front of a cluster of VNFCs to distribute the traffic flow.

In such case, the LB is deployed in front of a cluster of multiple VNFCs. Such
cluster can be managed by a seperate cluster manager, or can be managed just
by the LB,  which uses heartbeat to monitor each VNFC. When one of VNFCs fails,
the cluster manager should first exclude the failed VNFC from the cluster so that
the LB will re-route the traffic to the other VNFCs, and then the failed one should
be recovered. In the case when the LB is acting as the cluster manager, it is
the LB's responsibility to inform the VNFM to recover the failed VNFC if possible.


Case 2: LB used in front of a cluster of VMs to distribute traffic flow.

In this case, there exists a cluster manager(e.g. Pacemaker) to monitor and manage
the VMs in the cluster. The LB sits in front of the VM cluster so as to distribute
the traffic. When one of the VM fails, the cluster manager will detect that and will
be in charge of the recovery. The cluster manager will also exclude the failed VM
out of the cluster, so that the LB won't route traffic to the failed one.
 
In both two cases, the HA of the LB should also be considered.


3.2. Intra-VNF Communication

For stateful VNFs, data synchronization is necessary between the active and standby VMs.
The HA manager is responsible for handling VNFC failover, and do the assignment of the
active/standby states between the VNFCs of the VNF. Data synchronization can be handled
either by the HA manager or by the VNFC itself.

The state synchronization can happen as

- direct communication between the active and the standby VNFCs

- based on the information received from the HA manager on channel or messages using a common queue,

- it could be through a shared storage assigned to the whole VNF

- through the checkpointing of state information via underlying memory and/or
database checkpointing services to a separate VM and storage repository.
