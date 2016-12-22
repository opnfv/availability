==============================================================
Reach-thru Guest Monitoring and Services for High Availability
==============================================================
--------
Overview
--------

:author: Greg Waines
:organization: Wind River Systems
:organization: OPNFV - High Availability
:status: Draft
:date: March 2017
:revision: 1.2

:abstract: This document provides an overview of a set 
   of new optional capabilities where the OpenStack Cloud messages 
   into the Guest VMs in order to provide improved Availability 
   of the hosted VMs.  These new capabilities include: enabling the
   detection of and recovery from internal VM faults, enabling 
   Guest VMs to gracefully handle and provide loss-of-service warnings
   to cloud adminstrative operations on the VM, enabling vCPU scaling 
   to deal with high load scenarios and providing a simple out-of-band 
   messaging service to prevent scenarios such as split brain.


.. sectnum::

.. contents:: Table of Contents



Introduction
============

   This document provides an overview and rationale of a set 
   of new capabilities where the OpenStack Cloud messages 
   into the Guest VMs in order to provide improved Availability 
   of the hosted VMs.  
   
   These new capabilities specifically include: 

        - VM Heartbeating and Health Checking
        - VM Event Notification and Feedback
        - VM Peer State Notification and Messaging
        - VM Resource Scaling

   All of these capabilities leverage Host-to-Guest Messaging 
   Interfaces / APIs which are built on a messaging service between the 
   OpenStack Host and the Guest VM that uses a simple low-bandwidth 
   datagram messaging capability in the hypervisor and therefore has no 
   requirements on OpenStack Networking, and is available very early in 
   the lifecycle of the VM.



Messaging Layer
===============

   The Host-to-Guest messaging APIs used by the services discussed 
   in this document use a JSON-formatted application messaging layer 
   on top of a ‘virtio serial device’ between QEMU on the OpenStack Host 
   and the Guest VM.  JSON formatting provides a simple, humanly readable 
   messaging format which can be easily parsed and formatted using any 
   high level programming language being used in the Guest VM (e.g. C/C++, 
   Python, Java, etc.).  Use of the ‘virtio serial device’ provides a 
   simple, direct communication channel between host and guest which is
   independent of the Guest’s L2/L3 networking. 

   The upper layer JSON messaging format is actually structured as a
   hierarchical JSON format containing a Base JSON Message Layer and an
   Application JSON Message Layer:

        - the Base Layer provides the ability to multiplex different groups
          of message types on top of a single ‘virtio serial device’ 
          e.g.
     
           + heartbeating and healthchecks,
           + server group messaging, 
           + resource scaling, 
     
          and
     
        - the Application Layer provides the specific message types and
          fields of a particular group of message types.



VM Heartbeating and Health Checking
===================================

   Normally OpenStack monitoring of the health of a Guest VM is limited
   to a black-box approach of simply monitoring the presence of the
   QEMU/KVM PID containing the VM.

   VM Heartbeating and Health Checking provides a heartbeat service to monitor 
   the health of guest application(s) within a VM running under the OpenStack 
   Cloud.  Loss of heartbeat or a failed health check status will result in a 
   fault event being reported to OPNFV's DOCTOR infrastructure for alarm identification, 
   impact analysis and reporting.  This would then enable VNF Managers listening to 
   OPNFV's DOCTOR External Alarm Reporting through Telemetry's AODH, to initiate 
   any required fault recovery actions.

   .. image:: OPNFV_HA_Guest_APIs-Overview_HLD-Guest_Heartbeat-FIGURE-1.png

   The VM Heartbeating and Health Checking functionality is enabled on 
   a VM through a new flavor extraspec indicating that the VM supports 
   and wants to enable Guest Heartbeating.  Nova Compute uses this extraspec
   to setup the required 'virtio serial device' for Host-to-Guest messaging,
   on the QEMU/KVM instance created for the VM.

   A daemon within the Guest VM will register with the OpenStack Guest 
   Heartbeat Service on the compute node to initiate the heartbeating.  
   Part of that registration process is the specification of a heartbeat interval 
   in msecs.  Also part of that registration process is a 'suggested' default 
   corrective action, from the Guest, for a failed/unhealthy VM condition.  Guest 
   requested corrective actions can include simply logging the event, rebooting the 
   VM or stopping the VM.  However it should be noted that this is only a suggestion,
   and final fault recovery actions are the responsiblilty/authority of the VNF Managers.

   Guest heartbeat works on a challenge response model.  The OpenStack
   Guest Heartbeat Service on the compute node will challenge the registered 
   Guest VM daemon with a message each interval.  The registered Guest VM daemon 
   must respond prior to the next interval with a message indicating good health.  
   If the OpenStack Host does not receive a valid response, or if the response 
   specifies that the VM is in ill health, then a fault event for the Guest VM 
   is reported to the OpenStack Guest Heartbeat Service on the controller node which
   will report the event to OPNFV's DOCTOR.

   The registered Guest VM daemon's response to the challenge can be as simple 
   as just immediately responding with OK.  This alone allows for detection of
   a failed or hung QEMU/KVM instance, or a failure of the OS within the VM to 
   schedule the registered Guest VM's daemon or failure to route basic IO within
   the Guest VM.

   However the registered Guest VM daemon's response to the challenge can be more 
   complex, running anything from a quick simple sanity check of the health of 
   applications running in the Guest VM, to a more thorough audit of the 
   application state and data.  In either case returning the status of the 
   health check enables the OpenStack host to detect and report the event in order
   to initiate recovery from application level errors or failures within the Guest VM.



VM Event Notification and Feedback
==================================

   VM Event Notification and Feedback provides application(s) within a Guest VM running 
   under OpenStack, the ability to receive notification of and provide feedback on actions 
   about to be performed against the VM.  For example, on notifications, the guest application 
   within the VM can take this opportunity to cleanly shut down or transfer its 
   service to a peer VM, and / or provide feedback that loss of application service 
   may occur due to state of the Guest VM or its peer standby Guest VM.



VM Event Notifications
----------------------

   A Nova Proxy entity takes over the public REST APIs of the core Nova component.
   Nova Proxy intercepts all messages to core Nova, and for a subset of those 
   Nova messages first notifies the guest of the upcoming request, prior to forwarding
   the message on to Nova for actual execution of the command.

   A registered daemon running in the Guest VM is used as a conduit for
   notifications of VM lifecycle events being taken by OpenStack that
   will impact this VM.  Reboots, pause/resume and migrations are examples of
   the types of events a Guest VM can be notified of via the Nova Proxy mechanism.
   The full list of events for which notifications are supported is found below.

        - stop
        - reboot
        - pause
        - unpause
        - suspend
        - resume
        - resize
        - live-migrate
        - cold-migrate

   .. image:: OPNFV_HA_Guest_APIs-Overview_HLD-Guest_Notifications-FIGURE-2.png

   Notifications are an opportunity for the VM to take preparatory actions
   in anticipation of the forthcoming event. For example,

        - A reboot or stop notification might allow the application to stop
          accepting transactions and cleanly wrap up existing transactions, 
	  and/or gracefully close files; ensuring no data loss,

   The registered daemon in the Guest VM will receive all events.  If
   an event is not of interest, it should return immediately with a
   successful return code.  Notifications are subject to configurable timeouts.  
   Timeouts are specified by the registering daemon in the Guest VM.

   While notification handlers are running, the event will be delayed.
   If the timeout is reached, the event will be allowed to proceed.



VM Event Feedback
-----------------

   In addition to notifications, Nova Proxy also provides the opportunity for 
   the VM to provide feedback on any proposed event.  Currently the feedback
   supported is either 'ok' or 'lossOfService-warning'.  Nova Proxy will first 
   send out a Feedback Request, and then depending on the received feedback either
   abort the command (i.e. if 'lossOfService-warning' received) or send the Notification
   of the event to the VM (i.e. if 'ok' received), followed by forwarding the command
   on to Nova for execution.
   
   The Feedback request is subject to a configurable timeout.  The timeout is specified 
   when the daemon in the Guest VM registers with compute services on the host.  If the 
   VM fails to provide feedback within the timeout it is assumed to have no feedback with
   respect to the proposed action.

   An example of an application scenario where a feedback of lossOfService-warning
   would be returned on a feedback request:

       - an active-standby application deployment (1:1), where the active
         provides lossOfService-warning feedback on a shutdown or pause or ... 
	 due to its peer standby is not ready or synchronized.

   A feedback request handler should generally not take any action beyond returning its
   feedback.  Instead any preparatory actions should be done on the event notification that follows.

   The OpenStack Cloud is not required to respect the feedback from the VM.  Any feedback warnings 
   may be ignored in certain scenarios.  For REST API / CLI commands from operators, Nova Proxy 
   will not send a feedback request if a '--force' option is present; i.e. only a Notification
   message will be sent to the VM, before forwarding the command request on to core Nova.



VM Peer State Notification and Messaging
========================================

   Server Group State Notification and Messaging is a service to provide 
   simple low-bandwidth datagram messaging and notifications for servers that 
   are part of the same server group.  This messaging channel is available 
   regardless of whether IP networking is functional within the server, and 
   it requires no knowledge within the server about the other members of the group.
   
   The service provides three types of messaging:

        - Broadcast: this allows a server to send a datagram (size of up to 3050 bytes)
          to all other servers within the server group.
        - Notification: this provides servers with information about changes to the
          (Nova) state of other servers within the server group.
        - Status: this allows a server to query the current (Nova) state of all servers within
          the server group (including itself).
        
   A Server Group Messaging entity on both the controller node and the compute nodes 
   manage the routing of of VM-to-VM messages through the platform, leveraging Nova
   to determine Server Group membership and compute node locations of VMs.  The Server
   Group Messaging entity on the controller also listens to Nova VM state change notifications
   and querys VM state data from Nova, in order to provide the VM query and notification 
   functionality of this service.

   .. image:: OPNFV_HA_Guest_APIs-Overview_HLD-Peer_Messaging-FIGURE-3.png

   This service is not intended for high bandwidth or low-latency operations.  It
   is best-effort, not reliable.  Applications should do end-to-end acks and
   retries if they care about reliability.
   
   This service provides building block type capabilities for the Guest VMs that
   contribute to higher availability of the VMs in the Guest VM Server Group.  Notifications 
   of VM Status changes potentially provide a faster and more accurate notification
   of failed peer VMs than traditional peer VM monitoring over Tenant Networks.  While 
   the Broadcast Messaging mechanism provides an out-of-band messaging mechanism to
   monitor and control a peer VM under fault conditions; e.g. providing the ability to 
   avoid potential split brain scenarios between 1:1 VMs when faults in Tenant 
   Networking occur.



VM Resource Scaling
===================

   The VM Resource Scaling is a service to allow the OpenStack Cloud to scale the 
   capacity of a single guest server up and down on demand.
   
   Current supported scaling operation is CPU scaling in a 'dedicated' or cpu-pinned
   scenario.  I.e. VM CPU Resource Scaling is not supported for a VM using shared cpu 
   policy.
   
   Resources can be scaled up/down via NOVA extensions / patches to Nova Api, Conductor, 
   Scheduler and Compute.  A new flavor extraspec is introduced which defines the minimum 
   number of vCPUs; where the vCPU attribute of the flavor is interpreted as the max vCPUs.  
   ::

      nova flavor-create <flavor-name> <id> 4096 500 4      // i.e. 4 vcpus  (max)
      nova flavor-key <flavor-name> set hw:min_vcpus=2      // i.e. 2 vcpus  (min)

      ...

      nova boot --flavor <flavor-name> ... <instance-name>

      ...

      nova scale <instance-name> cpu down
      nova scale <instance-name> cpu down
      ...
      nova scale <instance-name> cpu up
      ...

   
   The scaling up or down can be managead by VNF Managers calling the REST API for the
   new 'nova scale ...' commands or thru a new Resource Scaling Policy introduced to 
   Heat by this activity.

   Nova scheduler changes are such that a VM is always initially scheduled and launched 
   with the max vCPUs and then scaled down and up again from there.  Scaling down involves 
   Nova Compute triggering the hypervisor to adjust the vCPU affinity so that the underlying 
   physical CPU can be freed up for use by other VMs and notifying the Guest VM (via the VM 
   Resource Scaling API) to re-assign any proceses affined specifically to the vCPU being 
   removed and to offline the specified vCPU in the kernel.  On scaling up, assuming the 
   resources are available, Nova compute will trigger the hypervisor to allocate a physical 
   CPU, associate it with the guest server and adjust the vCPU affinity to use the new physical 
   CPU, and notify the Guest VM (via the VM Resource Scaling API) to online the specified vCPU 
   in the kernel and optionally adjust any process vCPU affinities to use the new vCPU.
   
   .. image:: OPNFV_HA_Guest_APIs-Overview_HLD-Resource_Scaling-FIGURE-4.png

   The VM Resource Scaling Messaging Interface / API provides Guest VM notification of 
   the newly available or removed resources so that the Guest VM can perform any required
   kernel re-configurations as well as perform any application-level re-distribution of
   loads across the re-sized resources.

   It is possible for the scale-up operation to fail if the compute node has
   already allocated all of its resources to other guests.  If this happens,
   the system will not do any automatic migration to try to free up resources.
   Manual action will be required to free up resources.

   The behaviour of a scaled-down server/instance/VM during various nova 
   operations is as follows:
   
        - live migration: server remains scaled-down
        - pause/unpause: server remains scaled-down
        - stop/start: server remains scaled-down
        - evacuation: server remains scaled-down
        - rebuild: server remains scaled-down
        - automatic restart on crash: server remains scaled-down
        - cold migration: server reverts to max vcpus
        - resize: server reverts to max vcpus for the new flavor
   
   If a snapshot is taken of a scaled-down server, a new server booting the
   snapshot will start with the number of vCPUs specified by the flavor.

   Scaling can also be realized thru Scaling extensions in HEAT, allowing resources 
   to be automatically triggered based on Ceilometer statistics.  

   VM CPU Resource Scaling contributes to higher availability by:
   
        - providing a mechanism for VMs to use a minimal amount of vCPUs, freeing up physical 
          CPUs for other VMs,
        - providing a mechanism for VMs run with a low number of CPUs for steady state loads,
          but increase CPUs under heavy load in order to maintain service.



Conclusion
==========

   The Reach-thru Guest Monitoring and Services described in this document
   leverage Host-to-Guest messaging to provide a number of extended capabilities 
   that improve the Availability of the hosted VMs.  These new capabilities 
   enable detection of and recovery from internal VM faults, enables the Guest 
   VMs to gracefully handle or even reject cloud adminstrative operations on the 
   VM, provides a simple out-of-band messaging service to prevent scenarios such 
   as split brain and provides vCPU scaling to deal with high load scenarios in 
   a VM.

