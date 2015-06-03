===============
2.0 Hardware HA
===============



The hardware HA can be solved by several legacy HA schemes. However, when considering the NFV scenarios, 
hardware failure will cause collateral damage to not only virtual infrastructures but also services running on it. 

The redundant architecture and automatic failover for the hardware are required for the NFV scenario. At the same time, the fault detection and report of HW failure from the hardware to VIM, VNFM and Orchestrator is necessary for the HA approaches in OPNFV. A sample fault table can be found in the Doctor project. https://wiki.opnfv.org/doctor/faults. All the critical hardware failures should be reported to the VIM within 1s. 

.. (MT2) Should we keep the 50ms here? Other places have been modified to <1sec, e.g. for SAL 1.

.. (fq2) agree with 1s

Other warnings for the hardware should also be reported to the VIM in a timely manner.

*********************
General Requirements:
*********************

.. (MT) Are these general requirements or just for the servers?

.. (fq)  I think these should be the general requirements. not just the server.

Hardware Failure should be reported to the hypervisor and the VIM instead of directly reporting to the NF in the traditional 

.. (MT) I would assume that this is OK if no guest was impacted, if there was a guest impact I think the VIM etc should know about the issue; in any case logging the failure and its correction would be still important 

.. (fq) It seems the hardware failure detection message should send to VIM, shall we delete the hypervisor part?

.. (MT) The reason I asked the question whether this is about the servers was the hypervisor. I agree to remove this from the genaral requirement.

.. (Yifei)  Shall we take VIM user (VNFM & NFVO) into consideration? As some of the messages should be send to VIM user. 

.. (fq) yifei, I am a little bit confused, do you mean the Hardware send messages directly to VIM user? I myself think this may not be possible?

.. (Yifei) Yes, ur right, they should be sent to VIM first.

.. (MT) I agree, they should be sent to the VIM, the hypervisor can only be conditional because it may not be relevant as in a general requirement or may be dead with the HW.

.. (fq) Agree. I have delete the hypervisor part so that it is not a general requirement.

Hardware failure detection message should be sent to the VIM within 1s (may require realtime features in openstack?)? 

.. (fq) We may need some discussion about the time constraints? including failure detection time, VNF failover time, warning for abnormal situations. A table might be needed to clearify these. Different level of VNF may require differnent failover time.

.. (MT) I agree. A VNF that manages its own availability with "built-in" redundancy wouldn't really care whether it's 1s or 1min because it would detect the failure and do the failover at the VNF level. But if the availability is managed by the VIM and VNFM then this time becomes critical.

.. (joe) VIM can only rescue or migrate the VM onto anther host in case of hardware failure. The VNF should have being rescalready finish the failover before the failed/fault VM  ued or migrated. VIM's responisbility is to keep the number of alive VM instances required by VNF, even for auto scaling, but not to replacethe VNF failover.That's why hardware failure dection message for VIM is not so time sensitive, because VM creation is often a slow task compared to failover(Althoug a lot of technology to accelerate the VM generation speed or use spare VM pool ).

.. (fq) Yes. But here we just mean failure detection, not rescue or migration of the VM. I mean the hardware and NFVI failure should be reported to the VIM and the VNF in a timely manner, then the VNF can do the failover, and the VIM can do the migration and rescue afterwards. 

.. (bb) There is confusion regarding time span within which hardware failure should be reported to VIM. In 2nd paragraph(of Hardware HA), it has been mentioned as; "within 50ms" and in this point it is "1s". 

.. (fq) I try to modify the 50ms to 1s.

.. (chayi) hard for openstack 

VNF failover time < 1s

.. (MT) Indeed, it's not designed for that

.. (MT) Do the "hardware failure detection message" and the "alarm of hardware failure" refer to the same notification? It may be better to speak about hardware failure detection (and reporting) time. 

.. (fq) I have made the modification. see if it makes sense to you now.

.. (MT) Based on the definition section I think you are talking about these threshold alarms only, because a failure is also an abnormal situation, but you want to detect it within a second

.. (fq) Actually, I want to define Alarm as messages that might lead to failure in the near future, for example, a high tempreture, or maybe a prediction of failure. These alarm maybe important, but they do not need to be answered and solved within seconds.

Alarms for abnormal situations and performance decrease (i.e. overuse of cpu) should be raised to the VIM within 1min(?).  Alarm thresholds should be detected and the alarm delivered to the VIM within 1min. A certain threshold can be set for such notification (may require realtime extension for openstack?)

.. (MT) There should be possible to set some threshold at which the notification should be triggered and probably ceilometer is not reliable enough to deliver such notifications since it has no real-time requirement nor it is expected to be lossless.

.. (fq) modification made.

.. (MT) agree with the realtime extension part :-)

.. (MT) Considering the modified definitions can we say that: Alarm conditions should be detected and the alarm delivered to the VIM within 1min?

This effectively result in two requirements: one on the detection and one on the delivery mechanism.

.. (fq) Agree. I have made the modification.

Direct notice (maybe notification is a better word) from the hardware to some specific VNFs is possible.  Such notice should be within 1s. (may require extension of hypervisor?)

.. (Yifei) As before I do not think it is needed to send HW fault/failure to VNF. For it is different from traditional interated NF, all the lifecycle of VNF is managed by VNFM. 

.. (joe) the HW fault/failure to VNF is required directly for VNF failover purpose. For example, memory or nic failure should be noticed by VNF ASAP, so that the service can be taken over and handled correctly by another VNF instance.

.. (YY) In what case HW failure to VNF directly?Next is my understanding,may be not correct. If cpu/memory fails hostOS may be crashed at the same time the failure occured then no notification could be send to anywhere. If it is not crashed in some well managed smp OS, and if we use cpu-pinning to VM, the vm guestOS may be crashed. If cpu-pinning is not applied to VM, the hypervisor can continue scheduling the VMs on the server just like over-allocation mode. Another point, to accelerate the failover, the failure should be sent to standby service entity not the failed one. The standby vm should not be in same server because of anti-affinity scheme. How can "direct notice" apply?

.. (joe) not all HW fault leads to the VNF will be crushed. For example, the nic can not send packet as usual, then it'll affect the service, but the VNF is still running. 

Periodical update of hardware running conditions (operational state?) to the NFVI and VIM is required for further operation, which may include fault prediction, failure analysis, and etc.. Such info should be updated every 60s10 min(?) ( may require extensions of the detection point of celiometer?)

Maybe 10 min is too long. As far as I know, Zabbix which is used by Doctor can achieve 60s.

.. (fq) change the constraint to 60s

Transparent failover is required once the failure of storage and network hw happens

.. (MT2) I think this applies primarily to storage, network hardware and maybe some controllers, which also run in some type of redundancy e.g. active/active or active/standby. For compute, we need redundancy, but it's more of the spare concept to replace any failed compute in the cluster (e.g. N+1). In this context the failover doesn't mean the recovery of a state, it only means replacing the failed HW with a healthy one in the initial state and that's not transparent at the HW level at least, i.e. the host is not brought up with the same identiy as the failed one.

.. (fq) agree. I have made some modification. I wonder what controller do you mean? is it SDN controller?

.. (MT3) Yes, SDN, storage controllers. I don't know if any of the OpenStack controllers would also have such requirement, e.g. Ironic

Hardware should support SNMP and IPMI for centralized management, monitor and control.

.. (MT) Is it expected for _all_ hardware? 

As general requirement should we add that the hardware should allow for centralized management and control? Maybe we could be even more specific e.g. what protocol should be supported. 

.. (fq) I agree. as far as I know, the protocol we use for hardware include SNMP and IPMI.

.. (MT) OK, we can start with those as minimum requirement, i.e. HW should support at least them. Also I think the Ironic project in OpenStack manages the HW and also supports these.  I was thinking maybe it could also be used for the HW management although that's not the general goal of Ironic as far as I know. 

***************************
Network plane Requirements:
***************************

The hardware should provide redundancy architecture for the network plane.
Failures of the network plane should be reported to the VIM within 1s 50ms

.. (MT) Do you mean the failure of the entire network plane?
.. (fq) no, I mean the failure of the network connection of a certain HW, or a VNF.

Link failure, Qos reduction, Link congestion, and etc. will all trigger network plane automatic failover.

********************
Power supply system:
********************

Redundancy architecture for the power supply system.
Fault of the power supply system should be reported to the VIM within 1s 50ms
Failure of the power supply will trigure automatic failover of the system

***************
Cooling system:
***************

Redundancy architecture for the cooling system.
Fault of the cooling system should be reported to the VIM within 1s 50ms
Failure of the cooling systme will triggerure automatic failover of the system

***********
Disk Array:
***********

Redundancy architecture for the power supply, cooling, and interface is required
Fault of the disk array should be reported to the VIM within 1s 50ms
Failure of the power supply, cooling system and interface of the disk array will triggerure automatic failover of the system
support for protected cache after an unexpected power loss 

********
Servers:
********

Support precise timming with accuracy higher than 4.6ppm

.. (MT2) Should we have time synchronization requirements in the other parts? I.e. having NTP in control nodes or even in all hosts
