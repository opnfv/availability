.. image:: opnfv-logo.png 
  :height: 40
  :width: 200
  :alt: OPNFV
  :align: left


===================================================
1  Overall Principle for High Availability in NFV
===================================================

The ultimate goal for the High Availability schema is to provide high
availability to the upper layer services.

High availability is provided by the following steps once a failure happens:

    Step 1: failover of services once failure happens and service is out of work

    Step 2: Recovery of failed parts in each layer.

******************************************
1.1 Framework for High Availability in NFV
******************************************

Framework for Carrier Grade High availability:

A layered approach to availability is required for the following reasons:

* fault isolation
* fault tolerance
* fault recovery

Among the OPNFV projects the OPNFV-HA project's focus is on requirements related
to service high availability. This is complemented by other projects such as the
OPNFV - Doctor project, whose focus is reporting and management of faults along
with maintenance, the OPNFV-Escalator project that considers the upgrade of the
NFVI and VIM, or the OPNFV-Multisite that adds geographical redundancy to the
picture.

A layered approach allows the definition of failure domains (e.g., the
networking hardware, the distributed storage system, etc.). If possible, a fault
shall be handled at the layer (failure domain) where it occurs. If a failure
cannot be handled at its corresponding layer, the next higher layer needs to be
able to handle it. In no case, shall a failure cause cascading failures at other
layers.

The layers are:


+---------------------------+-------------------------------------+
+       Service             +     End customer visible service    |
+===========================+=====================================+
+     Application           +     VNF's, VNFC's                   |
+---------------------------+-------------------------------------+
+      NFVI/VIM             +     Infrastructure, VIM, VNFM, VM   |
+---------------------------+-------------------------------------+
+      Hardware             +     Servers, COTS platforms         |
+---------------------------+-------------------------------------+

The following document describes the various layers and how they need to
address high availability.

**************
1.2 Definitons
**************

Reference from the ETSI NFV doc.

**Availability:** Availability of an item to be in a state to perform a required
function at a given instant of time or at any instant of time within a given
time interval, assuming that the external resources, if required, are provided.

**Accessibility:** It is the ability of a service to access (physical) resources
necessary to provide that service. If the target service satisfies the minimum
level of accessibility, it is possible to provide this service to end users.

**Admission control:** It is the administrative decision (e.g. by operator's
policy) to actually provide a service. In order to provide a more stable and
reliable service, admission control may require better performance and/or
additional resources than the minimum requirement. Failure: deviation of the
delivered service from fulfilling the system function.

**Fault:** adjudged or hypothesized cause of an error

**Service availability:** service availability of <Service X> is the long-term
average of the ratio of aggregate time between interruptions to scheduled
service time of <ServiceX> (expressed as a percentage) on a user-to-user basis.
The time between interruptions is categorized as Available (Up time) using the
availability criteria as defined by the parameter thresholds that are relevant
for <Service X>.

Accoring to the ETSI GS NFV-REL 001 V1.1.1 (2015-01) document service
availability in the context of NFV is defined as End-to-End Service availability

.. (MT) The relevant parts in NFV-REL defines SA as:

Service Availability refers to the End-to-End Service Availability which
includes all the elements in the end-to-end service (VNFs and infrastructure
components) with the exception of the customer terminal. This is a customer
facing (end user) availability definition and it is the result of accessibility
and #admission control (see their respective definitions above).

Service Availability=total service available time/
                     (total service available time + total restoration time)

**Service continuity:** Continuous delivery of service in conformance with
service's functional and behavioural specification and SLA requirements,
both in the control and data planes, for any initiated transaction or session
until its full completion even in the events of intervening exceptions or
anomalies, whether scheduled or unscheduled, malicious, intentional
or unintentional.

The relevant parts in NFV-REL:
The basic property of service continuity is that the same service is provided
during VNF scaling in/out operations, or when the VNF offering that service
needs to be relocated to another site due to an anomaly event
(e.g. CPU overload, hardware failure or security threat).

**Service failover:** when the instance providing a service/VNF becomes
unavailable due to fault or failure, another instance will (automatically) take
over the service, and this whole process is transparent to the user. It is
possible that an entire VNF instance becomes unavailble while providing its
service.

.. (MT) I think the service or an instance of it is a logical entity on its own and the service availability and continuity is with respect to this logical entity. For examlpe if a HTTP server serves a given URL, the HTTP server is the provider while that URL is the service it is providing. As long as I have an HTTP server running and serving this URL I have the service available. But no matter how many HTTP servers I'm running if they are not assigned to serve the URL, then it is not available. Unfortunately in the ETSI NFV documents there's not a clear distinction between the service and the provider logical entities. The distinction is more on the level of the different incarnations of the provider entity, i.e. VNF and its instances or VNFC and its instances. I don't know if I'm clear enough and to what extent we should go into this, but I tried to modify the definition along these lines.  Now regarding the user perception and whether it's automatic I agreed that we want it automatic and seemless for the user, but I don't think that this is part of the failover definition. If it's done manually or if the user detects it it's still a failover. It's just not seemless. Requiring it being automatic and seemless should be in the requirement section as appropriate. 

.. (fq) Agree.

**Service failover time:** Service failover is when the instance providing a
service becomes unavailable due to a fault or a failure and another healthy
instance takes over in providing the service. In the HA context this should be
an automatic action and this whole process should be transparent to the user.
It is possible that an entire VNF instance becomes unavailble while providing
its service.

.. (MT) Aligned with the above I would say that the serice failover time is the time from the moment of detecting the failure of the instance providing the service until the service is provided again by a new instance.

.. (fq) So in such definition, the time duration for the failure of the service=failure detection time+service failover time. Am I correct?

.. (bb) I feel, it is;  "time duration for failover of the service = failure detection time + service failover time".
.. (MT) I would say that the "failure detection time" + "service failover time" = "service outage time" or actually we defined it below as the "service recovery time" .  To reduce the outage we probably can't do much with the "service failover time", it is whatever is needed to perform the failover procedure, so it's tied to the implementation. It's somewhat "given". We may have more control over the detection time as that depends on the frequency of the health-check/heartbeat as this is often configurable.

.. (fq) Got it. Agree.

**Failure detection:** If a failure is detected, the failure must be identified
to the component responsible for correction.

.. (MT) I would rather say "failure detection" as the fault is not detectable until it becomes a failure, even then we may not know where the actual fault is. We only know what failed due to the fault. E.g. we can detect the memory leak, something may crash due to it, but it's much more difficult to figure out where the fault is, i.e. the bug in the software. 

.. (MT) Also I think failures may be detected by different entities in the system, e.g. it could be a monitoring entity, a watchdog, the hypervisor, the VNF itself or a VNF tryng to use the services of a failed VNF. For me all these are failure detections regardless whether they are reported to the VNF. I think from an HA perspective what's important is the error report API(s) that entities should use if they detect a failure they are not in charge of correcting.
.. (fq) Agree. I modify the definition.

**Failure detection time:** Failure detection time is the time interval from the
moment the failure occurs till it is reported as a detected failure.

**Alarm:** Alarms are notifications (not queried) that are activated in response
to an event, a set of conditions, or the state of an inventory object.  They
also require attention from an entity external to the reporting entity (if not
then the entity should cope with it and not raise the alarm).

.. (MT) According to NFV-INF 004: Alarms are notifications (not queried) that are activated in response to an event, a set of conditions, or the state of an inventory object.  I would add also that they also require attention from an entity external to the reporting entity (if not then the entity should cope with it and not raise the alarm).

**Alarm threshold condition detection:** Alarm threshold condition is detected
by the component responsible for it.  The component periodically evaluates the
condition associated with the alarm and if the threshold is reached, it
generates an alarm on the approprite channel, which in turn delivers it to the
entity(ies) responsible, such as the VIM.

.. (fq) I don't think the VNF need to know all the alarm. so I use VIM as the terminal point for the alarm detection

.. (MT) The same way as for the faults/failures, I don't think it's the receiving end that is important but the generatitng end and that it has the right and appropriate channel to communicate the alarm. But I have the impression that you are focusing on a particular type of alarm (i.e. threshold alarm) and not alarms in general.

.. (fq) Yes, I actully have the threshold alarm in my mind when I wrote this. So I think VIM might be the right receiving end for these alarm. I agree with your ideas about the right channel. I am just not sure whether we should put this part in a high lever perspective or we should define some details. After all OPNFV is an opensource project and we don't want it to be like standarization projects in ETSI. But I agree for the definition part we should have a high level and abstract definition for these, and then we can specify the detail channels in the API definition.

.. (MT) I tried to modify accordingly. Pls check. I think when it comes to the receiver we don't need to be specific from the detection perspective as usually there is a well-known notification channel that the management entity if it exists would listen to. The alarm detection does not require this entity, it just states that something is wrong and someone should deal with it hence the alarm.

**Alarm threshold detection time:** the threshold time interval between the
metrics exceeding the threshold and the alarm been detected.

.. (MT) I assume you are focusing on these threshold alarms, and not alarms in general.
.. (MT) Here similar to the failover time, we may have some control over the detection time (i.e. shorten the evaluation period), but may not on the delivery time.
.. (MT2) I changed "condition" to "threshold" to make it clearer as failure is a "condition" too :-)

**Service recovery:** The restoration of the service state after the instance of
a service/VNF is unavailable due to fault or failure or manual interuption.

.. (MT) I think the service recovery is the restoration of the state in which the required function is provided 

**Service recovery time:** Service recovery time is the time interval from the
occurrence of an abnormal event (e.g. failure, manual interruption of service,
etc.) until recovery of the service.

.. (MT) in NFV-REL: Service recovery time is the time interval from the occurrence of an abnormal event (e.g. failure, manual interruption of service, etc.) until recovery of the service.

**SAL:** Service Availability Level

************************
1.3 Overall requirements
************************

Service availability shall be considered with respect to the delivery of end to
end services.

* There should be no single point of failure in the NFV framework
* All resiliency mechanisms shall be designed for a multi-vendor environment,
  where for example the NFVI, NFV-MANO, and VNFs may be supplied by different
  vendors.
* Resiliency related information shall always be explicitly specified and
  communicated using the reference interfaces (including policies/templates) of
  the NFV framework.

*********************
1.4 Time requirements
*********************

The time requirements below are examples in order to break out of the failure
detection times considering the service recovery times presented as examples for
the different service availability levels in the ETSI GS NFV-REL 001 V1.1.1
(2015-01) document.

The table below maps failure modes to example failure detection times.

+------------------------------------------------------------+---------------+
|Failure Mode                                                |  Time         |
+============================================================+===============+
|Failure detection of HW                                     |  <1s          |
+------------------------------------------------------------+---------------+
|Failure detection of virtual resource                       |  <1s          |
+------------------------------------------------------------+---------------+
|Alarm threshold detection                                   |  <1min        |
+------------------------------------------------------------+---------------+
|Failure detection over of SAL 1                             |  <1s          |
+------------------------------------------------------------+---------------+
|Recovery of SAL 1                                           |  5-6s         |
+------------------------------------------------------------+---------------+
|Failure detectionover of SAL 2                              |  <5s          |
+------------------------------------------------------------+---------------+
|Recovery of SAL 2                                           |  10-15s       |
+------------------------------------------------------------+---------------+
|Failure detectionover of SAL 3                              |  <10s         |
+------------------------------------------------------------+---------------+
|Recovery of SAL 3                                           |  20-25s       |
+------------------------------------------------------------+---------------+


===============
2  Hardware HA
===============

The hardware HA can be solved by several legacy HA schemes. However, when
considering the NFV scenarios, a hardware failure will cause collateral damage to
not only to the services but also virtual infrastructure running on it.

A redundant architecture and automatic failover for the hardware are required
for the NFV scenario. At the same time, the fault detection and report of HW
failure from the hardware to VIM, VNFM and if necessary the Orchestrator to achieve HA in OPNFV. A
sample fault table can be found in the Doctor project. (https://wiki.opnfv.org/doctor/faults)
All the critical hardware failures should be reported to the VIM within 1s.

.. (MT2) Should we keep the 50ms here? Other places have been modified to <1sec, e.g. for SAL 1.

.. (fq2) agree with 1s

Other warnings for the hardware should also be reported to the VIM in a
timely manner.

*****************************
2.1 General Requirements
*****************************

.. (MT) Are these general requirements or just for the servers?

.. (fq)  I think these should be the general requirements. not just the server.

* Hardware Failures should be reported to the hypervisor and the VIM.
* Hardware Failures should not be directly reported to the VNF as in the traditional ATCA
  architecture.
* Hardware failure detection message should be sent to the VIM within a specified period of time,
  based on the SAL as defined in Section 1.
* Alarm thresholds should be detected and the alarm delivered to the VIM within 1min. A certain
  threshold can be set for such notification.
* Direct notification from the hardware to some specific VNF should be possible.
  Such notification should be within 1s.
* Periodical update of hardware running conditions (operational state?) to the
  NFVI and VIM is required for further operation, which may include fault
  prediction, failure analysis, and etc.. Such info should be updated every 60s
* Transparent failover is required once the failure of storage and network
  hardware happens.
* Hardware should support SNMP and IPMI for centralized management, monitoring and
  control.

.. (MT) I would assume that this is OK if no guest was impacted, if there was a guest impact I think the VIM etc should know about the issue; in any case logging the failure and its correction would be still important 
.. (fq) It seems the hardware failure detection message should send to VIM, shall we delete the hypervisor part?
.. (MT) The reason I asked the question whether this is about the servers was the hypervisor. I agree to remove this from the genaral requirement.
.. (Yifei)  Shall we take VIM user (VNFM & NFVO) into consideration? As some of the messages should be send to VIM user. 
.. (fq) yifei, I am a little bit confused, do you mean the Hardware send messages directly to VIM user? I myself think this may not be possible?
.. (Yifei) Yes, ur right, they should be sent to VIM first.
.. (MT) I agree, they should be sent to the VIM, the hypervisor can only be conditional because it may not be relevant as in a general requirement or may be dead with the HW.
.. (fq) Agree. I have delete the hypervisor part so that it is not a general requirement.
.. may require realtime features in openstack

.. (fq) We may need some discussion about the time constraints? including failure detection time, VNF failover time, warning for abnormal situations. A table might be needed to clearify these. Different level of VNF may require differnent failover time.

.. (MT) I agree. A VNF that manages its own availability with "built-in" redundancy wouldn't really care whether it's 1s or 1min because it would detect the failure and do the failover at the VNF level. But if the availability is managed by the VIM and VNFM then this time becomes critical.

.. (joe) VIM can only rescue or migrate the VM onto anther host in case of hardware failure. The VNF should have being rescalready finish the failover before the failed/fault VM  ued or migrated. VIM's responisbility is to keep the number of alive VM instances required by VNF, even for auto scaling, but not to replacethe VNF failover.That's why hardware failure dection message for VIM is not so time sensitive, because VM creation is often a slow task compared to failover(Althoug a lot of technology to accelerate the VM generation speed or use spare VM pool ).

.. (fq) Yes. But here we just mean failure detection, not rescue or migration of the VM. I mean the hardware and NFVI failure should be reported to the VIM and the VNF in a timely manner, then the VNF can do the failover, and the VIM can do the migration and rescue afterwards. 

.. (bb) There is confusion regarding time span within which hardware failure should be reported to VIM. In 2nd paragraph(of Hardware HA), it has been mentioned as; "within 50ms" and in this point it is "1s". 

.. (fq) I try to modify the 50ms to 1s.

.. (chayi) hard for openstack 

.. VNF failover time < 1s

.. (MT) Indeed, it's not designed for that

.. (MT) Do the "hardware failure detection message" and the "alarm of hardware failure" refer to the same notification? It may be better to speak about hardware failure detection (and reporting) time. 

.. (fq) I have made the modification. see if it makes sense to you now.

.. (MT) Based on the definition section I think you are talking about these threshold alarms only, because a failure is also an abnormal situation, but you want to detect it within a second

.. (fq) Actually, I want to define Alarm as messages that might lead to failure in the near future, for example, a high tempreture, or maybe a prediction of failure. These alarm maybe important, but they do not need to be answered and solved within seconds.

.. Alarms for abnormal situations and performance decrease (i.e. overuse of cpu)
.. should be raised to the VIM within 1min(?).  


.. (MT) There should be possible to set some threshold at which the notification should be triggered and probably ceilometer is not reliable enough to deliver such notifications since it has no real-time requirement nor it is expected to be lossless.

.. (fq) modification made.

.. (MT) agree with the realtime extension part :-)

.. (MT) Considering the modified definitions can we say that: Alarm conditions should be detected and the alarm delivered to the VIM within 1min?

.. This effectively result in two requirements: one on the detection and one on the
.. delivery mechanism.

.. (fq) Agree. I have made the modification.



.. In the meantime, I see the discussion of
.. this requirement is still open.

.. (Yifei) As before I do not think it is needed to send HW fault/failure to VNF. For it is different from traditional interated NF, all the lifecycle of VNF is managed by VNFM. 

.. (joe) the HW fault/failure to VNF is required directly for VNF failover purpose. For example, memory or nic failure should be noticed by VNF ASAP, so that the service can be taken over and handled correctly by another VNF instance.

.. (YY) In what case HW failure to VNF directly?Next is my understanding,may be not correct. If cpu/memory fails hostOS may be crashed at the same time the failure occured then no notification could be send to anywhere. If it is not crashed in some well managed smp OS, and if we use cpu-pinning to VM, the vm guestOS may be crashed. If cpu-pinning is not applied to VM, the hypervisor can continue scheduling the VMs on the server just like over-allocation mode. Another point, to accelerate the failover, the failure should be sent to standby service entity not the failed one. The standby vm should not be in same server because of anti-affinity scheme. How can "direct notice" apply?

.. (joe) not all HW fault leads to the VNF will be crushed. For example, the nic can not send packet as usual, then it'll affect the service, but the VNF is still running. 


.. Maybe 10 min is too long. As far as I know, Zabbix which is used by Doctor can
.. achieve 60s.

.. (fq) change the constraint to 60s

.. (MT2) I think this applies primarily to storage, network hardware and maybe some controllers, which also run in some type of redundancy e.g. active/active or active/standby. For compute, we need redundancy, but it's more of the spare concept to replace any failed compute in the cluster (e.g. N+1). In this context the failover doesn't mean the recovery of a state, it only means replacing the failed HW with a healthy one in the initial state and that's not transparent at the HW level at least, i.e. the host is not brought up with the same identiy as the failed one.

.. (fq) agree. I have made some modification. I wonder what controller do you mean? is it SDN controller?

.. (MT3) Yes, SDN, storage controllers. I don't know if any of the OpenStack controllers would also have such requirement, e.g. Ironic



.. (MT) Is it expected for _all_ hardware? 

.. (YY) As general requirement should we add that the hardware should allow for
.. centralized management and control? Maybe we could be even more specific
.. e.g. what protocol should be supported.

.. (fq) I agree. as far as I know, the protocol we use for hardware include SNMP and IPMI.

.. (MT) OK, we can start with those as minimum requirement, i.e. HW should support at least them. Also I think the Ironic project in OpenStack manages the HW and also supports these.  I was thinking maybe it could also be used for the HW management although that's not the general goal of Ironic as far as I know. 

*********************************
2.2  Network plane Requirements
*********************************

* The hardware should provide a redundant architecture for the network plane.
* Failures of the network plane should be reported to the VIM within 1s.
* QoS should be used to protect against link congestion.

.. (MT) Do you mean the failure of the entire network plane?
.. (fq) no, I mean the failure of the network connection of a certain HW, or a VNF.

**************************
2.3  Power supply system
**************************

* The power supply architecture should be redundant at the server and site level.
* Fault of the power supply system should be reported to the VIM within 1s.
* Failure of a power supply will trigure automatic failover to the redundant supply.

*********************
2.4  Cooling system
*********************

* The architecture of the cooling system should be redundant.
* Fault of the cooling system should be reported to the VIM within 1s
* Failure of the cooling systme will trigger automatic failover of the system

***************
2.5 Disk Array
***************

* The architecture for the disk array should be redundant.
* Fault of the disk array should be reported to the VIM within 1s
* Failure of the the disk array will trigger automatic failover of the system
  support for protected cache after an unexpected power loss.

* Data shall be stored redundantly in the storage backend
    (e.g., by means of RAID across disks.)
* Upon failures of storage hardware components (e.g., disks services, storage
  nodes) automatic repair mechanisms (re-build/re-balance of data) shall be
  triggered automatically.
* Centralized storage arrays shall consist of redundant hardware

*************
2.6 Servers
*************

* Support precise timing with accuracy higher than 4.6ppm

.. (MT2) Should we have time synchronization requirements in the other parts? I.e. having NTP in control nodes or even in all hosts


====================================================
3  Virtualization Facilities (Host OS, Hypervisor)
====================================================

**********************************************************
3.1 Requirements on Host OS and Hypervisor and Storage 
**********************************************************

Requirements:
==============

- The hypervisor should support distributed HA mechanism
- Hypervisor should detect the failure of the VM. Failure of the VM should be reported to
  the VIM within 1s
- The hypervisor should report (and if possible log) its failure and recovery action.
  and the destination to whom they are reported should be configurable.
- The hypervisor should support VM migration
- The hypervisor should provide isolation for VMs, so that VMs running on the same
  hardware do not impact each other.
- The host OS should provide sufficient process isolation so that VMs running on
  the same hardware do not impact each other.
- The hypervisor should record the VM information regularly and provide logs of
  VM actions for future diagnoses.
- The NFVI should maintain the number of VMs provided to the VNF in the face of failures.
  I.e. the failed VM instances should be replaced by new VM instances

************************************
3.2 Requirements on Middlewares
************************************

Requirements:
==============

- It should be possible to detect and automatically recover from hypervisor failures
  without the involvement of the VIM
- Failure of the hypervisor should be reported to the VIM within 1s
- Notifications about the state of the (distributed) storage backends shall be send to the
  VIM (in-synch/healthy, re-balancing/re-building, degraded).
- Process of VIM runing on the compute node should be monitored, and failure of it should
  be notified to the VIM within 1s
- Fault detection and reporting capability. There should be middlewares supporting in-band
  reporting of HW failure to VIM.
- Storage data path traffic shall be redundant and fail over within 1 second on link
  failures.
- Large deployments using distributed software-based storage shall separate storage and
  compute nodes (non-hyperconverged deployment).
- Distributed software-based storage services shall be deployed redundantly.
- Data shall be stored redundantly in distributed storage backends.
- Upon failures of storage services, automatic repair mechanisms (re-build/re-balance of
  data) shall be triggered automatically.
- The storage backend shall support geo-redundancy.

=============================================
4 Virtual Infrastructure HA 每 Requirements
=============================================

This section is written with the goal to ensure that there is alignment with
Section 4.2 of the ETSI/NFV REL-001 document.

Key reference requirements from ETSI/NFV document:
===================================================

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

**************
4.1 Compute
**************

VM including CPU, memory and ephemeral disk

.. (Yifei) Including noca-compute fq) What do you mean? Yifei) I mean nova-
.. (compute is important enough for us to define some requirement about it.
.. (IJ)(Nova-compute is important, but implementation specific, this should be
.. requirements focused.

Requirements:
==============

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
.. host. Failing to do so 每 when the perceived disconnection is due to some
.. transient or partial failure 每 the evacuation might lead into two identical
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
.. For larger system 每 replication of data across multiple storage nodes.  Processes controlling the storage nodes must also be replicated, such that there is no single point of failure.
.. Block storage supported by a clustered files system is required.
.. Should be tranparent to the storage user

************
4.2 Network
************

4.2.1 Virtual network
========================

Requirements:
--------------
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

4.2.2 vSwitch 
===============

Requirements:
--------------

* Monitoring and health of vSwitch processes is required.
* The vSwitch must adapt to changes in network topology and automatically
  support recovery modes in a transparent manner.

4.2.3 Link Redundancy
=========================

Requirements:
--------------

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



============================
5     VIM High availability
============================
The VIM in the NFV reference architecture  contains all the control nodes of OpenStack, SDN controllers
and hardware controllers. It manages the NFVI according to the instructions/requests of the VNFM and
NFVO and reports them back about the NFVI status. To guarantee the high availability of the VIM is
a basic requirement of the OPNFV platform. Also the VIM should provide some mechanism for VNFs to achieve
their own high availability.


*******************************************
5.1 Architecture requirement of VIM HA
*******************************************

The architecture of the control nodes should avoid any single point of failure and the management
network plane which connects the control nodes should also be redundant. Services of the control nodes
which are stateless like nova-API, glance-API etc. should be redundant but without data synchronization.
Stateful services like MySQL, Rabbit MQ, SDN controller should provide complex redundancy policies.
Cloud of different scale may also require different HA policies.

Requirement:
=============
- In small scale scenario active-standby redundancy policy would be acceptable.

- In large scale scenario all stateful services like database, message queue, SDN controller
  should be deployed in cluster mode which support N-way, N+M active-standby redundancy.

- In large scale scenario all stateless services like nova-api, glance-api etc. should be deployed
  in all active mode.

- Load balance nodes which introduced for all active and N+M mode should also avoid the single point
  of failure.

- All control node servers shall have at least two network ports to connect to different networks
  plane. These ports shall work in bonding manner.

- Any failures of services in the redundant pairs should be detected and switch over should be carried out
  automatically in less than 5 seconds totally.

- Status of services must be monitored.

******************************************************
5.2 Fault detection and alarm requirement of VIM
******************************************************


Redundant architecture can provide function continuity for the VIM. For maintenance considerations
all failures in the VIM should be detected and notifications should be triggered to NFVO, VNFM and other
VIM consumers.

Requirement:
=============

- All hardware failures of control nodes should be detected and relevant alarms should be triggered.
  OSS, NFVO, VNFM and other VIM consumers can subscribe these alarms.

- Software on control nodes like OpenStack or ODL should be monitored by the clustering software
  at process level and alarms should be triggered when exceptions are detected.

- Software on compute nodes like OpenStack/nova agents, ovs should be monitored by watchdog. When
  exceptions are detected the software should be restored automatically and alarms should be triggered.

- Software on storage nodes like Ceph, should be monitored by watchdog. When
  exceptions are detected the software should be restored automatically and alarms should be triggered.

- All alarm indicators should include: Failure time, Failure location, Failure type, Failure level.

- The VIM should provide an interface through which consumers can subscribe to alarms and notifications.

- All alarms and notifications should be kept for future inquiry in VIM, ageing policy of these records
  should be configurable.

- VIM should distinguish between the failure of the compute node and the failure of the host HW.

- VIM should be able to publish the health status of the compute node to NFV MANO.

*******************************************
5.3 HA mechanism of VIM provided for VNFs
*******************************************

When VNFs deploy their HA scheme, they usually require from underlying resource to provide some mechanism.
This is similar to the hardware watchdog in the traditional network devices. Also virtualization
introduces some other requirements like affinity and anti-affinity with respect to the allocation of the
different virtual resources.

Requirement:
============

- VIM should provide the ability to configure HA functions like watchdog timers,
  redundant network ports and etc. These HA functions should be properly tagged and exposed to
  VNF and VNFM with standard APIs.

- VIM should provide anti-affinity scheme for VNF to deploy redundant service on different level of
  aggregation of resource.

- VIM should be able to deploy classified virtual resources to VNFs following the SAL description in VNFD.

- VIM should provide data collection to calculate the HA related metrics for VNFs.

- VIM should support the VNF/VNFM to initiate the operation of resources of the NFVI, such as repair/reboot.

- VIM should correlate the failures detected on collocated virtual resources to identify latent faults in
  HW and virtualization facilities

- VIM should be able to disallow the live migration of VMs and when it is allowed it should be possible
  to specify the tolerated interruption time.

- VIM should be able to restrict the simultaneous migration of VMs hosting a given VNF.

- VIM should provide the APIs to trigger scale in/out to VNFM/VNF.

- When scheduler of the VIM use the Active/active HA scheme, multiple scheduler instances must not create
  a race condition

- VIM should be able to trigger the evacuation of the VMs before bringing the host down
  when *maintenance mode* is set for the compute host.

- VIM should configure Consoleauth in active/active HA mode, and should store the token in database.

- VIM should replace a failed VM with a new VM and this new VM should start in the same initial state
  as the failed VM.

- VIM should support policies to prioritize a certain VNF.

*********************
5.4 SDN controller
*********************

SDN controller: Distributed or Centralized

Requriements:
==============
- In centralized model SDN controller must be deployed as redundant pairs.

- In distributed model, mastership election must determine which node is in overall control.

- For distributed model, VNF should not be aware of HA of controller. That is it is a - logically centralized
  system for NBI(Northbound Interface).

- Event notification is required as section 5.2 mentioned.

=======================
6 VNF High Availability
=======================


************************
6.1 Service Availability
************************

In the context of NFV, Service Availability refers to the End-to-End (E2E) Service
Availability which includes all the elements in the end-to-end service (VNFs and
infrastructure components) with the exception of the customer terminal such as
handsets, computers, modems, etc. The service availability requirements for NFV
should be the same as those for legacy systems (for the same service).

Service Availability =
total service available time /
(total service available time + total service recovery time)

The service recovery time among others depends on the number of redundant resources
provisioned and/or instantiated that can be used for restoring the service.

In the E2E relation a Network Service is available only of all the necessary
Network Functions are available and interconnected appropriately to collaborate
according to the NF chain.

General Service Availability Requirements
=========================================

* We need to be able to define the E2E (V)NF chain based on which the E2E availability
  requirements can be decomposed into requirements applicable to individual VNFs and
  their interconnections
* The interconnection of the VNFs should be logical and be maintained by the NFVI with
  guaranteed characteristics, e.g. in case of failure the connection should be
  restored within the acceptable tolerance time
* These characteristics should be maintained in VM migration, failovers and switchover,
  scale in/out, etc. scenarios
* It should be possible to prioritize the different network services and their VNFs.
  These priorities should be used when pre-emption policies are applied due to
  resource shortage for example.
* VIM should support policies to prioritize a certain VNF.
* VIM should be able to provide classified virtual resources to VNFs in different SAL


6.1.1 Service Availability Classification Levels
=================================================


The [ETSI-NFV-REL_] defined three Service Availability Levels
(SAL) are classified in Table 1. They are based on the relevant ITU-T recommendations
and reflect the service types and the customer agreements a network operator should
consider.

.. [ETSI-NFV-REL] `ETSI GS NFV-REL 001 V1.1.1 (2015-01) <http://www.etsi.org/deliver/etsi_gs/NFV-REL/001_099/001/01.01.01_60/gs_NFV-REL001v010101p.pdf>`_


*Table 1: Service Availability classification levels*

+-------------+-----------------+-----------------------+---------------------+
|SAL Type     | Customer Type   |  Service/Function     |   Notes             |
+=============+=================+=======================+=====================+
|Level 1      | Network Operator|  * Intra-carrier      |   Sub-levels within |
|             | Control Traffic |    engineering        |   Level 1 may be    |
|             |                 |    traffic            |   created by the    |
|             | Government/     |  * Emergency          |   Network Operator  |
|             | Regulatory      |    telecommunication  |   depending on      |
|             | Emergency       |    service (emergency |   Customer demands  |
|             | Services        |    response, emergency|   E.g.:             |
|             |                 |    dispatch)          |                     |
|             |                 |  * Critical Network   |   * 1A - Control;   |
|             |                 |    Infrastructure     |   * 1B - Real-time; |
|             |                 |    Functions (e.g     |   * 1C - Data;      |
|             |                 |    VoLTE functions    |                     |
|             |                 |    DNS Servers,etc.)  |   May require 1+1   |
|             |                 |                       |   Redundancy with   |
|             |                 |                       |   Instantaneous     |
|             |                 |                       |   Switchover        |
+-------------+-----------------+-----------------------+---------------------+
|Level 2      | Enterprise and/ |  * VPN                |  Sub-levels within  |
|             | or large scale  |  * Real-time traffic  |  Level 2 may be     |
|             | customers       |    (Voice and video)  |  created  by the    |
|             | (e.g.           |  * Network            |  Network Operator   |
|             | Corporations,   |    Infrastructure     |  depending on       |
|             | University)     |    Functions          |  Customer demands.  |
|             |                 |    supporting Level   |  E.g.:              |
|             | Network         |    2 services (e.g.   |                     |
|             | Operators       |    VPN servers,       |  * 2A - VPN;        |
|             | (Tier1/2/3)     |    Corporate Web/     |  * 2B - Real-time;  |
|             | service traffic |    Mail servers)      |  * 2C - Data;       |
|             |                 |                       |                     |
|             |                 |                       |  May require 1:1    |
|             |                 |                       |  Redundancy with    |
|             |                 |                       |  Fast (maybe        |
|             |                 |                       |  Instantaneous)     |
|             |                 |                       |  Switchover         |
+-------------+-----------------+-----------------------+---------------------+
|Level 3      | General Consumer|  * Data traffic       |  While this is      |
|             | Public and ISP  |    (including voice   |  typically          |
|             | Traffic         |    and video traffic  |  considered to be   |
|             |                 |    provided by OTT)   |  "Best Effort"      |
|             |                 |  * Network            |  traffic, it is     |
|             |                 |    Infrastructure     |  expected that      |
|             |                 |    Functions          |  Network Operators  |
|             |                 |    supporting Level   |  will devote        |
|             |                 |    3 services         |  sufficient         |
|             |                 |                       |  resources to       |
|             |                 |                       |  assure             |
|             |                 |                       |  "satisfactory"     |
|             |                 |                       |  levels of          |
|             |                 |                       |  availability.      |
|             |                 |                       |  This level of      |
|             |                 |                       |  service may be     |
|             |                 |                       |  pre-empted by      |
|             |                 |                       |  those with         |
|             |                 |                       |  higher levels of   |
|             |                 |                       |  Service            |
|             |                 |                       |  Availability. May  |
|             |                 |                       |  require M+1        |
|             |                 |                       |  Redundancy with    |
|             |                 |                       |  Fast Switchover;   |
|             |                 |                       |  where M > 1 and    |
|             |                 |                       |  the value of M to  |
|             |                 |                       |  be determined by   |
|             |                 |                       |  further study      |
+-------------+-----------------+-----------------------+---------------------+

Requirements
-------------

* It shall be possible to define different service availability levels
* It shall be possible to classify the virtual resources for the different
  availability class levels
* The VIM shall provide a mechanism by which VNF-specific requirements
  can be mapped to NFVI-specific capabilities.

More specifically, the requirements and capabilities may or may not be made up of the
same KPI-like strings, but the cloud administrator must be able to configure which
HA-specific VNF requirements are satisfied by which HA-specific NFVI capabilities.



6.1.2 Metrics for Service Availability
======================================

The [ETSI-NFV-REL_] identifies four metrics relevant to service
availability:

* Failure recovery time,
* Failure impact fraction,
* Failure frequency, and
* Call drop rate.

6.1.2.1 Failure Recovery Time
---------------------------------

The failure recovery time is the time interval from the occurrence of an abnormal
event (e.g. failure, manual interruption of service, etc.) until the recovery of the
service regardless if it is a scheduled or unscheduled abnormal event. For the
unscheduled case, the recovery time includes the failure detection time and the
failure restoration time.
More specifically restoration also allows for a service recovery by the restart of
the failed provider(s) while failover implies that the service is recovered by a
redundant provider taking over the service. This provider may be a standby
(i.e. synchronizing the service state with the active provider) or a spare
(i.e. having no state information). Accordingly failover also means switchover, that
is, an orederly takeover of the service from the active provider by the standby/spare.

Requirements:
^^^^^^^^^^^^^^^

* It should be irrelevant whether the abnormal event is due to a scheduled or
  unscheduled operation or it is caused by a fault.
* Failure detection mechanisms should be available in the NFVI and configurable so
  that the target recovery times can be met
* Abnormal events should be logged and communicated (i.e. notifications and alarms as
  appropriate)

The TL-9000 forum has specified a service interruption time of 15 seconds as outage
for all traditional telecom system services. [ETSI-NFV-REL_]
recommends the setting of different thresholds for the different Service Availability
Levels. An example setting is given in the following table 2. Note that for all
Service Availability levels Real-time Services require the fastest recovery time.
Data services can tolerate longer recovery times. These recovery times are applicable
to the user plane. A failure in the control plane does not have to impact the user plane.
The main concern should be simultaneous failures in the control and user planes
as the user plane cannot typically recover without the control plane. However an HA
mechanism in VNF itself can further mitigate the risk. Note also that the impact on
the user plane depends on the control plane service experiencing the failure,
some of them are more critical than others.


*Table 2: Example service recovery times for the service availability levels*

+------------+-----------------+------------------------------------------+
|SAL         |  Service        |  Notes                                   |
|            |  Recovery       |                                          |
|            |  Time           |                                          |
|            |  Threshold      |                                          |
+============+=================+==========================================+
|1           | 5 - 6 seconds   | Recommendation: Redundant resources to be|
|            |                 | made available on-site to  ensure fast   |
|            |                 | recovery.                                |
+------------+-----------------+------------------------------------------+
|2           | 10 - 15 seconds | Recommendation: Redundant resources to be|
|            |                 | available as a mix of on-site and off-   |
|            |                 | site as appropriate.                     |
|            |                 |                                          |
|            |                 |  * On-site resources to be utilized for  |
|            |                 |    recovery of real-time services.       |
|            |                 |  * Off-site resources to be utilized for |
|            |                 |    recovery of data services.            |
+------------+-----------------+------------------------------------------+
|3           | 20 - 25 seconds | Recommendation: Redundant resources to be|
|            |                 | mostly available off-site. Real-time     |
|            |                 | services should be recovered before data |
|            |                 | services                                 |
+------------+-----------------+------------------------------------------+


6.1.2.2 Failure Impact Fraction
------------------------------------

The failure impact fraction is the maximum percentage of the capacity or user
population affected by a failure compared with the total capacity or the user
population supported by a service. It is directly associated with the failure impact
zone which is the set of resources/elements of the system to which the fault may
propagate.

Requirements:
^^^^^^^^^^^^^^^

* It should be possible to define the failure impact zone for all the elements of the
  system
* At the detection of a failure of an element, its failure impact zone must be
  isolated before the associated recovery mechanism is triggered
* If the isolation of the failure impact zone is unsuccessful the isolation should be
  attempted at the next higher level as soon as possible to prevent fault propagation.
* It should be possible to define different levels of failure impact zones with
  associated isolation and alarm generation policies
* It should be possible to limit the collocation of VMs to reduce the failure impact
  zone as well as to provide sufficient resources

6.1.2.3 Failure Frequency
---------------------------

Failure frequency is the number of failures in a certain period of time.

Requirements:
^^^^^^^^^^^^^^^^

* There should be a probation period for each failure impact zones within which
  failures are correlated.
* The threshold and the probation period for the failure impact zones should be
  configurable
* It should be possible to define failure escalation policies for the different
  failure impact zones


6.1.2.4 Call Drop Rate
------------------------

Call drop rate reflects service continuity as well as system reliability and
stability. The metric is inside the VNF and therefore is not specified further for
the NFV environment.

Requirements:
^^^^^^^^^^^^^^^^

* It shall be possible to specify for each service availability class the associated
  availability metrics and their thresholds
* It shall be possible to collect data for the defined metrics
* It shall be possible to delegate the enforcement of some thresholds to the NFVI
* Accordingly it shall be possible to request virtual resources with guaranteed
  characteristics, such as guaranteed latency between VMs (i.e. VNFCs), between a VM
  and storage, between VNFs


**********************
6.2 Service Continuity
**********************

The determining factor with respect to service continuity is the statefulness of the
VNF. If the VNF is stateless, there is no state information which needs to be
preserved to prevent the perception of service discontinuity in case of failure or
other disruptive events.
If the VNF is stateful, the NF has a service state which needs to be preserved
throughout such disruptive events in order to shield the service consumer from these
events and provide the perception of service continuity. A VNF may maintain this state
internally or externally or a combination with or without the NFVI being aware of the
purpose of the stored data.

Requirements:
===============

* The NFVI should maintain the number of VMs provided to the VNF in the face of
  failures. I.e. the failed VM instances should be replaced by new VM instances
* It should be possible to specify whether the NFVI or the VNF/VNFM handles the
  service recovery and continuity
* If the VNF/VNFM handles the service recovery it should be able to receive error
  reports and/or detect failures in a timely manner.
* The VNF (i.e. between VNFCs) may have its own fault detection mechanism, which might
  be triggered prior to receiving the error report from the underlying NFVI therefore
  the NFVI/VIM should not attempt to preserve the state of a failing VM if not
  configured to do so
* The VNF/VNFM should be able to initiate the repair/reboot of resources of the VNFI
  (e.g. to recover from a fault persisting at the VNF level => failure impact zone
  escalation)
* It should be possible to disallow the live migration of VMs and when it is allowed
  it should be possible to specify the tolerated interruption time.
* It should be possible to restrict the simultaneous migration of VMs hosting a given
  VNF
* It should be possible to define under which circumstances the NFV-MANO in
  collaboration with the NFVI should provide error handling (e.g. VNF handles local
  recoveries while NFV-MANO handles geo-redundancy)
* The NFVI/VIM should provide virtual resource such as storage according to the needs
  of the VNF with the required guarantees (see virtual resource classification).
* The VNF shall be able to define the information to be stored on its associated
  virtual storage
* It should be possible to define HA requirements for the storage, its availability,
  accessibility, resilience options, i.e. the NFVI shall handle the failover for the
  storage.
* The NFVI shall handle the network/connectivity failures transparent to the VNFs
* The VNFs with different requirements should be able to coexist in the NFV Framework
* The scale in/out is triggered by the VNF (VNFM) towards the VIM (to be executed in
  the NFVI)
* It should be possible to define the metrics to monitor and the related thresholds
  that trigger the scale in/out operation
* Scale in operation should not jeopardize availability (managed by the VNF/VNFM),
  i.e. resources can only be removed one at a time with a period in between sufficient
  for the VNF to restore any required redundancy.



