2. Discussion for the General Issues for VNF HA schemes
====================================================

This section is intended to talk about some general issues in the VNF HA schemes.
In sections 1, the usecases of both stateful and stateless VNFs are discussed.
While in this section, we would like to discuss some specific issues
which are quite general for all the usecases proposed in the previous sections.

1.1. VNF External Interfacece

Regardless whether the VNF is stateful or stateless, all the VNFCs should act as
a union from the perspective of the outside world. That means all the VNFCs should share a common
interface where the outside modules (e.g., the other VNFs) can access the service
from. There could be multiple solutions for this share of IP interface. However,
all of this sharing and switching of IP address should be ignorant to the outside
modules.

There are several approaches for the VNFs to share the interfaces. A few of them
are listed as follows and will be discussed in detail. 

1) IP address of active/stand-by VM.

2) Load balancers for active/active use cases

Note that combinition of these two approaches is also feasible.

For active/standby VNFCs, the HA manager will manage a common IP address
to the active and standby VMs, so that they look as one instance from outside.
(The HA manager may not be aware of this, I.e. the address may be configured
and the active/standby state management is linked to the possession of the IP
address, i.e. the active VNFC claims it as part of becoming active.) Only the
active one possesses the IP address. And when failover happens, the standby
is set to be active and can take possession of the IP address to continue traffic
process.

..[MT] In general I would rather say that the IP address is managed by the HA
manager and not provided. But as a concrete use case "provide" works fine.
So it depends how you want to use this text.
..[fq] Agree, Thank you!

For active/active VNFCs, a load balancer is used in front of multiple active/active
VMs. A single virtual IP is used for all the VMs. The other VNFs connect to this 
VNF using the virtual IP, which is managed by the load balancer. The load balancer
will redirect packets arriving at that virtual IP to a VM hosting an active VNFC.
When one of the active VMs fails, a new VM instance will replace the failed one.
The load balancer will include the new instance into the cluster. In such a scheme,
the HA of the load balancer should also be considered.

..[MT] I think this use case needs to show also how the LB learns about the new VNFC.
Also we should distinguish VNFC and VM failures as VNFC failure wouldn't be detected
in the NFVI e.g. LB, so we need a resolution, an applicability comment at least.
..[fq] I think I have made a mistake here by saying the VNFC. Actually if the failure
only happens in VNFC, the VNFC should reboot itself rather than have a new VNFC taking
its place. So in this case, I think I should modify VNFC into VMs. And as you mentioned,
the NFVI level can hardly detect VNFC level failure.

..[MT] There could also be a combined case for the N+M redundancy, when there are N
actives but also M standbys at the VNF level.
..[fq] It could be. But I actually haven't see such a deployed case. So I am not sure
if I can discribe the schemes correctly:)

1.2. Intra-VNF Communication

For stateful VNFs, data synchronization is necessary between= the active and standby VMs.
The HA manager is responsible for handling VNFC failover, and do the assignment of the
active/standby states between the VNFCs of the VNF. Data synchronization can be handled
either by the HA manager or by the VNFC itself.

The state synchronization can happen as

- direct communication between the active and the standby VNFCs

- based on the information received from the HA manager on channel or messages using
a common queue,
..[MT] I don't understand the yellow inserted text
..[fq] Neither do I, actually. I think it is added by some one else and I can't make
out what it means as well:)

- it could be through a shared storage assigned to the whole VNF

- through in-memory database (checkpointing), when the database (checkpoint service)
takes care of the data replication.

