4.0 Virtual Infrastructure HA – Requirements:
=============================================

This section is written with the goal to ensure that there is alignment with
Section 4.2 of the ETSI/NFV REL-001 document.

Key reference requirements from ETSI/NFV document:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

[Req.4.2.12] On the NFVI level, there should be a transparent fail-over in the
case of for example compute, memory,storage or connectivity failures.

.. (fq) According to VNF part, the following bullet may be added:

* The virtual infrastructure should provide classified virtual resource for
  different SAL VNFs. Each class of the resources should have guaranteed
  performance metrics.

* Specific HA handling schemes for each classified virtual resource,
  e.g. recovery mechanisms, recovery priorities, migration options,
  should be defined.

* The NFVI should maintain the number of VMs provided to the VNF in the face of
  failures. I.e. the failed VM instances should be replaced by new VM instances.

.. (MT) this might be a requirement on the hypervisor and/or the
.. VIM. In this respect I wonder where the nova agent running on the compute node
.. belongs. Is it the VIM already or the Virtualization Facilities?  The reason I'm
.. asking is that together with the hypervisor they are in a unique position of
.. correlating different failures on the host that may be due to HW, OS or
.. hypervisor.

.. (fq) I agree this might be for the hypervisor part. The VNF (i.e.
.. between VNFCs) may have its own fault detection mechanism, which might be
.. triggered prior to receiving the error report from the underlying NFVI therefore
.. the NFVI/VIM should not attempt to preserve the state of a failing VM if not
.. configured to do so

4.1 Compute
===========

VM including CPU, memory and ephemeral disk

.. (Yifei) Including noca-compute fq) What do you mean? Yifei) I mean nova-
.. (compute is important enough for us to define some requirement about it.
.. (IJ)(Nova-compute is important, but implementation specific, this should be
.. requirements focused.

Requirements:

* Detection of failures must be sub 1 second.
* Recovery of a failed VM (VNF) must be automatic.  The recovery must re-launch
  the VM based on the required initial state defined in the VNFD.

.. (MT) I think this is the same essentially as the one brought over from the VNF part in the paragraph above, where I have the question also.
.. (Yifei) Different mechanisms should be defined according to the SLA of the service running on the VM.
.. (fq) What do you mean by failure detection? Do you mean hypervisor notice the failure and perform automatic recovery? or do you mean hypervisor notice the failure and inform VIM?
.. (fq) How to define the time limit for the failure detection? whether 1s is sufficient enough, or we should require for sometime less?

.. Requirements do have some dependency on the NFVI interface definitions that are
.. currently being defined by ETSI/NFV working groups.  Ongoing alignment will
.. be required.

* On evacuation, fencing of instances from an unreachable host is required.

.. orginal wording for above: Fencing instances of an unreachable host when evacuation happens.[GAP 10]

.. (YY) If a host is unreachable how to evacuate VMs on it? Fencing function may be moved toVIM part. 
.. (fq) copy from the Gap 10:

.. Safe VM evacuation has to be preceded by fencing (isolate, shut down) the failed
.. host. Failing to do so – when the perceived disconnection is due to some
.. transient or partial failure – the evacuation might lead into two identical
.. instances running together and having a dangerous conflict.

.. (unknown commenter) I agree it should be move to VIM part.
.. (IJ) Not clear what or if the above comment has been moved.

.. (Yifei) In OpenStack, evacuate means that "VMs whose storage is accessible from other nodes (e.g. shared storage) could be rebuilt and restarted on a target node", it is different from migration. link: https://wiki.openstack.org/wiki/Evacuate

* Resources of a migrated VM must be evacuated once the VM is
  migrated to a different compute node, placement policies must be preserved.
  For example during maintenance activities.

.. (MT) Do you mean maintenance of the compute node? In any case I think the evacuation should follow the palcement policy.
.. (fq) Yes. What placement policy do you mean?
.. (Yifei) e.g. keep the same scheduler hints as before, am I right ,@Maria?
.. (MT) Yes, the affinity, anti-affinity, etc
.. (fq) Got it. I am adding a requirement that the evacuation should follow the placement policy.
.. (fq) insert below.

* Failure detection of the VNF software process is required
  in order to detect the failure of the VNF sufficiently. Detection should be
  within less than 1 second.

.. ( may require interface extension)

.. (MT) What do youy mean by the VNF software process? Is it the application(s) running in the VM? If yes, Heat has such consideration already, but I'm only familiar with the first version which was cron job based and therefore the resolution was 1 minute. 
.. (fq) Yes, I mean the applications. 1 min might be too long I am afraid. I think this failure detection should be at least less than the failover time. Otherwise it does not make sense.
.. (I don't know if 50ms is sufficient enough, since we require the failover of the VNFs should be within 50ms, if the detection is longer than this, there is no meaning to do the detection)
.. (MT) Do you assume that the entire VM needs to be repaired in case of application failure? Also the question is whether there's a VM ready to failover to. It might be that OpenStack just starts to build the VM when the failover is triggere. If that's the case it can take minutes. If the VM exists then starting it still takes ~half a minute I think.
.. I think there's a need to have the VM images in shared storage otherwise there's an issue with migration and failover
.. (fq) I don't mean the recovery of the entire VM. I only mean the failover of the service. In our testing, we use an active /active VM, so it only takes less than 1s to do the failover. I understand the situation you said above. I wonder if we should set a time constraint for such failover? for me, I think such constraint should be less than second.
.. (Yifei) Maria, I cannot understand " If the VM exists then starting it still takes ~half a minute", would please explain it more detailed? Thank you.
.. (MT) As far as I know Heat rebuilds the VM from scratch as part of the failure recovery. Once the VM is rebuilt it's booted and only after that it can actualy provide service. This time till the VM is ready to serve can take 20-30sec after the VM is already reported as existing.
.. ([Yifei) ah, I see. Thank you so much!
.. (YY) As I understand, what heat provides is not what fuqiao wants here. To failover within 50ms/or 1s means two VMs are all running, in NFVI view there are two VMs running, but in application view one is master the other is standby. What I did not find above is how to monitoring application processes in VM? Tradictionally watchdog is applied to this task. In new version of Qemu watchdog is simulated with software but timeslot of watchdog could not be as narrow as hardware watchdog. I was told lower than 15s may cause fault action.
.. Do you mean this watchdog? https://libvirt.org/formatdomain.html#elementsWatchdog
.. (fq) Yes, Yuan Yue got my idea:)

.. 4.2 Storage dedicated section (new section 7).
.. (GK) please see dedicated section on storage below (Section 7)
.. Virtual disk and volumes for applications.
.. Storage related to NFVI must be redundant.
.. Requirements:
.. For small systems a small local redundant file system must be supported.
.. For larger system – replication of data across multiple storage nodes.  Processes controlling the storage nodes must also be replicated, such that there is no single point of failure.
.. Block storage supported by a clustered files system is required.
.. Should be tranparent to the storage user

4.2 Network
===========

Virtual network:
^^^^^^^^^^^^^^^^

Requirements:

* Redundant top of rack switches must be supported as part of the deployment.

.. (MT) Shouldn't this be a HW requirement?
.. (Yifei) Agree with Maria
.. (IJ) The ToR is not typically in the NFVI, that is why I put the ToR here.

* Static LAG must be supported to ensure sub 50ms detection and failover of
  redundant links between nodes. The distributed virtual router should
  support HA.

.. (Yifei) Add ?: Service provided by Network agents should be keeped availability and continuity. e.g. VRRP is used for L3 agent HA (keepalived or pacemaker)
.. (IJ) this is a requirements document.  Exclude the implementation details.  Added the requirement below

* Service provided by network agents should be highly available (L3 Agent, DHCP
  agent as examples)

* L3-agent, DHCP-agent should clean up network artifacts (IPs, Namespaces) from
  the database in case of failover.

vSwitch Requirements:
^^^^^^^^^^^^^^^^^^^^^

* Monitoring and health of vSwitch processes is required.
* The vSwitch must adapt to changes in network topology and automatically
  support recovery modes in a transparent manner.

Link Redundancy Requirements:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* The ability to manage redundant interfaces and support of LAG on the compute
  node is required.
* Support of LAG on all interfaces, internal platform control
  interfaces,internal platform storage interfaces, as well as interfaces
  connecting to provide networks.
* LACP is optional for dynamic management of LAG links
* Automated configuration LAG should support active/standby and
  balanced modes. Should adapt to changes in network topology and automatically
  support recovery modes in a transparent manner.
* In SR-IOV scenario, link redundancy could not be transparent, VM should have
  two ports directly connect to physical port on host. Then app may bind
  these two ports for HA.

.. (MT) Should we consider also load balancers? I'm not familiar with the LBaaS, but it seems to be key for the load distribution for the multi-VM VNFs. 
.. (YY) As I know LBaaS was not mature this time in openstack. Openstack does provide API for LBaaS,but it depend on LB entity and its plugin. We have not found any mature LB agent and LB entity in community. The LB inside VNF usually approached by VNF itsself.
.. (fq) I think LB should be taken into consideration as well. eventhough openstack now is not mature. This is how OPNFV is working, we work out requirement for our side, propose possible bp to openstack so that these features can be added in the future releases.
.. (YIfei) Agree. Because of it is not mature, there is possibility to find gap between OpenStack and our requirement. 
.. (MT) Agree. We may even influence how it matures ;-)
.. vlb, vFW are part of virtual resources?
.. (Yifei) From my side, network node.
.. (Yifei) If you mean LB or FW in NFVI, I do not think vXX is a suitable name as in OpenStack Neutron there are LBaas and FWaas. If you mean VNF, then you can call them vLB and vFW. However i do not think LBaas is the same as vLB, they are different use cases. What we need to consider should be LBaas and FWaas not vLB or vFW.
.. For more details about LBaas and FWaas, you can find on the wiki page of neutron...
.. (fq) Thank you for Yifei. I wonder what's the difference between vLB and LBaas. You mean they have different functions?
.. (IJ) LBaaS is good for enterprise - for Carrier applications won't higher data rates be needed and therefore a Load Balancer in a VNF is probably a better solution.