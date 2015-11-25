5, Multisite Scenario
====================================================

The Multisite scenario refers to the cases when VNFs are deployed on multiple VIMs.
There could be two typical usecases for such scenario.

One is in one DC, multiple openstack clouds are deployed. Taking into consideration that the
number of compute nodes in one openstack cloud are quite limited (nearly 100) for
both opensource and commercial product of openstack, multiple openstack clouds will
have to be deployed in the DC to manage thousands of servers. In such a DC, it should
be possible to deploy VNFs accross openstack clouds.
..(MT) Do we anticipate HA VNFs that require more than 100 VMs so that they need to
be deployed across DCs? Or the goal is to provide higher availability by deploying
across DCs?
..(fq) Here I just try to explain what multisite scenario means. I don't think HA should
be discussed in this scenario since as you said, we can not have 100 more VMs deployed
to be HA.

The other typical usecase is Geographic Redundancy (GR). GR deployment is to deal with more
catastrophic failures (flood, earthquake, propagating software fault, and etc.) of a single site.
In the Geographic redundancy usecase, VNFs are deployed in two sites, which are
geographically seperated and are deployed on NFVI managed by seperate VIM. When
such a catastrophic failure happens, the VNFs at the failed site can failover to
the redundant one so as to continue the service. Different VNFs may have specified
requirement of such failover. Some VNFs may need stateful failover, while others 
may just need their VMs restarted on the redundant site in their initial state. 
The first would create the overhead of state replication. The latter may still 
have state replication through the storage. Accordingly for storage we don't want
to loose any data, and for networking the NFs should be connected the same way as
they were in the original site. We probably want also to have the same number of
VMs on the redundant site coming up for the VNFs.
..(MT) I agree and this scenario is definitely not limited to HA VNFs. Thus there could
be different mechanisms for the state replication between the sites and from an HA
perspective in this case it is important that the replication mechanism does not degrade
the performance at normal behaviour.

The multisite scenario is also captured by the Multisite project, in which specific
requirements of openstack are also proposed for different usecases. However,
the multisite project mainly focuses on the requirement of these multisite
usecases on openstack. HA requirements are not necessarily the requirement
for the approaches discussed in multisite. While the HA project tries to
capture the HA requirements in these usecases.
https://gerrit.opnfv.org/gerrit/#/c/2123/
https://gerrit.opnfv.org/gerrit/#/c/1438/.


An architecure of stateful VNF with redundancy in the multisite scenario can be as
follows. Architecture for the other cases can be worked out accordingly.
https://wiki.opnfv.org/_detail/stateful_vnf_in_multisite_scenario.png?id=scenario_analysis_of_high_availability_in_nfv
..(MT) What is the relation of the VMs of a single site e.g. on the left hand side?
Do they collaborate? Do they protect each other? What makes the two VIMs independent
if they need to support that VNF and its VNFM? Could they be logically the same
VIM and wouldn't that be a better solution for the VNF?
..(fq) This is kind of architecture captureed from the multisite project's work.
One VM on the left site is acting as the active VNFC, and the other VM at the right
site is acting as the standby. I assume the two VIM are cooperate with each other
under the control of the orchestrator. I am also thinking that if the two VMs contrled
by one VIM would be a better solution. But apparently that is not the scenario for
multisite, cause they are thinking multisite means you have multi openstack.


Below listed the additinal labor and extra requirements of multisite comparing with
the basic usecases.

1, specific network support for the active/standby or active/active VNFs across VIM.

In the multisite scenario, instances constructing the VNFs can be placed across VIM.
This will introduce extra network support requirement. For example, heartbeat between
active/standby VMs placed across VIM requires overlay L2 network. The IP address used
for VNF to connect with other VNFs should be able to be floating across VIM as well. 

2, in the multisite scenario, a logical instance of VNFM should be put on multiple
VIM to manage the instances of VNFs placed across the VIM. 

3, in the VM failure scenarios, recovery of failed VM requires interface between
VNFM and the VIM. In the multisite scenario, the VNFM should have knowledge of
which VIM it should communicate with so as to recover the failed VNF.