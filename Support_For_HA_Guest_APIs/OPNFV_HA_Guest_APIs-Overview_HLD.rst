=========================
Support for HA Guest APIs
=========================
--------
Overview
--------

:author: Greg Waines, Ian Jolliffe
:organization: Wind River Systems
:organization: OPNFV - High Availability
:status: Draft
:date: January 2017
:revision: 1.0

:abstract: This document provides an overview of a set of 
   Messaging Interfaces / APIs between an OpenStack Cloud and 
   its Guest VMs that improves the Availability of the hosted VMs.
   The Messaging Interfaces / APIs enable detection of and recovery
   from internal VM faults, enables the Guest VMs to gracefully handle
   or even reject cloud adminstrative operations on the VM, enables vCPU
   scaling to deal with high load scenarios and provides a simple out-of-band
   messaging service to prevent scenarios such as split brain.
   
.. sectnum::

.. contents:: Table of Contents



Introduction
============

   This document provides an overview and rationale of a set of Messaging 
   Interfaces / APIs between an OpenStack Cloud and its Guest VMs that 
   improves the availability of the hosted VM.

   The Messaging Interfaces / APIs provide the following functionality:

        - VM Heartbeating and Health Checking
        - VM Event Notification and Acknowledgement
        - VM Peer State Notification and Messaging
        - VM Resource Scaling

   All of these Messaging Interfaces / APIs are built on a messaging
   service between the OpenStack Host and the Guest VM that uses a simple 
   low-bandwidth datagram messaging capability in the hypervisor and 
   therefore has no requirements on OpenStack Networking, and is available 
   very early in the lifecycle of the VM.



VM Heartbeating and Health Checking
===================================

   Normally OpenStack monitoring of the health of a Guest VM is limited
   to a black-box approach of simply monitoring the presence of the
   QEMU/KVM PID containing the VM.

   This API provides a heartbeat service to monitor the health of guest 
   application(s) within a VM running under the OpenStack Cloud.  Loss 
   of heartbeat or a failed health check status will result in a corrective 
   action being taken against the VM.  The heartbeat interval (in msecs) and 
   the corrective action is specified by the VM.  Requested corrective actions 
   can include logging the event, rebooting the VM or stopping the VM.

   A daemon within the Guest VM will register with the OpenStack Guest 
   Heartbeat Service on the compute node.  Part of that registration process 
   is the specification of a heartbeat interval (in msecs) and a default 
   corrective action for a failed/unhealthy VM.  

   Guest heartbeat works on a challenge response model.  The OpenStack
   Guest Heartbeat Service on the compute node will challenge the registered 
   Guest VM daemon with a message each interval.  The registered Guest VM daemon 
   must respond prior to the next interval with a message indicating good health.  
   If the OpenStack Host does not receive a valid response, or if the response 
   specifies that the VM is in ill health, then the corrective action is taken.

   The registered Guest VM daemon's response to the challenge can be as simple 
   as just immediately responding with OK.  This alone allows for detection of
   a failed or hung QEMU/KVM instance, or a failure of the OS within the VM to 
   schedule the registered Guest VM's daemon or failure to route basic IO within
   the Guest VM.

   However the registered Guest VM daemon's response to the challenge can be more 
   complex, running anything from a quick simple sanity check of the health of 
   applications running in the Guest VM, to a more thorough audit of the 
   application state and data.  In either case returning the status of the 
   health check and optionally a corrective action that overrides the default 
   corrective action specified at registration.  This enables the OpenStack host
   to detect and assist in recovering from application level errors or failures
   within the Guest VM.



VM Event Notification and Acknowledgement
=========================================

   This API provides application(s) within a Guest VM running under OpenStack, 
   the ability to receive notification of and vote to accept or reject actions 
   about to be performed against the VM.  On notifications, the guest application 
   within the VM can take this opportunity to cleanly shut down or transfer its 
   service to a peer VM.


VM Event Notifications
----------------------

   A registered daemon running in the Guest VM can be used as a conduit for
   notifications of VM lifecycle events being taken by OpenStack that
   will impact this VM.  Reboots, pause/resume and migrations are examples of
   the types of events a Guest VM can be notified of.  Depending on the event, a
   vote on the event may be offered before a notification is sent.  Notifications
   may precede the event, follow it or both.  The full list of events and
   notifications is found below.

        - stop
        - reboot
        - pause
        - unpause
        - suspend
        - resume
        - resize
        - live-migrate
        - cold-migrate

   Notifications are an opportunity for the VM to take preparatory actions
   in anticipation of the forthcoming event, or recovery actions after
   the event has completed.  A few examples

        - A reboot or stop notification might allow the application to stop
          accepting transactions and cleanly wrap up existing transactions, 
	  and/or gracefully close files; ensuring no data loss,
        - A 'resume' notification after a suspend might trigger a time
          adjustment,
        - Pre and post migrate notifications might trigger the application
          to de-register and then re-register with a network load balancer.

   The registered daemon in the Guest VM will receive all events.  If
   an event is not of interest, it should return immediately with a
   successful return code.  Notifications are subject to configurable timeouts.  
   Timeouts are specified by the registering daemon in the Guest VM.

   While pre-notification handlers are running, the event will be delayed.
   If the timeout is reached, the event will be allowed to proceed.

   While post-notification handlers are running, or waiting to be run,
   the OpenStack Cloud will not be able to declare the action complete.
   Post-notifications are subject to a timeout as well.  If the timeout is 
   reached, the event will be declared complete.


VM Event Acknowledgements / Voting
----------------------------------

   In addition to notifications, there is also an opportunity for the VM
   to vote on any proposed event.  Voting precedes all notifications,
   and offers the VM a chance to reject the event that the OpenStack Cloud 
   wishes to initiate.  

   Voting is subject to a configurable timeout.  The timeout is specified 
   when the daemon in the Guest VM registers with compute services on the 
   host.  If the VM fails to vote within the timeout it is assumed to have 
   agreed with the proposed action.

   Rejecting an event should be the exception, not the rule, reserved for
   cases when the VM is handling an exceptionally sensitive operation,
   as well as a slow operation that can't complete in the notification timeout.
   An example

       - an active-standby application deployment (1:1), where the active
         rejects a shutdown or pause or ... due to its peer standby is not
         ready or synchronized.

   A vote handler should generally not take any action beyond returning its
   vote.  Instead save your actions for the notification that follows.

   The OpenStack Cloud is not required to offer a vote.  Voting may be
   bypassed on certain recovery scenarios.



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
   
   Resources can be scaled up/down via NOVA extensions.  A new flavor extraspec is
   introduced which defines the minimum number of vCPUs; where the vCPU attribute 
   of the flavor is interpreted as the max vCPUs.  
   
   The VM Resource Scaling Messaging Interface / API provides Guest VM notification of 
   the newly available or removed resources so that the Guest VM can perform any required
   kernel re-configurations as well as perform any application-level re-distribution of
   loads across the re-sized resources.

   A VM is always initially scheduled and launched with the max vCPUs and then scaled 
   down and up again from there.  Scaling down involves the hypervisor adusting the 
   vCPU affinity so that the underlying physical CPU can be freed up for use by other
   VMs and notifying the Guest VM (via the VM Resource Scaling API) to re-assign 
   any proceses affined specifically to the vCPU being removed and to offline the 
   specified vCPU in the kernel.  On scaling up, assuming the resources are available,
   the hypervisor will allocate a physical CPU, associate it with the guest server and
   adjust the vCPU affinity to use the new physical CPU, and notify the Guest VM (via
   the VM Resource Scaling API) to online the specified vCPU in the kernel and optionally
   adjust any process vCPU affinities to use the new vCPU.
   
   It is possible for the scale-up operation to fail if the compute node has
   already allocated all of its resources to other guests.  If this happens,
   the system will not do any automatic migration to try to free up resources.
   Manual action will be required to free up resources.

   The behaviour of a scaled-down server during various nova operations is as
   follows:
   
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



Messaging Layer
===============

   The Host-to-Guest Server Group Messaging API is a message-based API 
   using a JSON-formatted application messaging layer on top of a 
   ‘virtio serial device’ between QEMU on the OpenStack Host and the 
   Guest VM.  JSON formatting provides a simple, humanly readable 
   messaging format which can be easily parsed and formatted using any 
   high level programming language being used in the Guest VM (e.g. C, 
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



Conclusion
==========

   The HA Guest APIs between an OpenStack Cloud and its Guest VMs provide 
   a number of extended capabilities that improve the Availability of the 
   hosted VMs.  The HA Guest APIs enable detection of and recovery from 
   internal VM faults, enables the Guest VMs to gracefully handle or even reject 
   cloud adminstrative operations on the VM, provides a simple out-of-band messaging 
   service to prevent scenarios such as split brain and provides vCPU scaling to deal 
   with high load scenarios in a VM.

