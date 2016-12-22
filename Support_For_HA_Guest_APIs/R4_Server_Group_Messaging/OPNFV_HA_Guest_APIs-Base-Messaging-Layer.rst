=========================
Support for HA Guest APIs
=========================
----------------------------------
Base Host-to-Guest Messaging Layer
----------------------------------

:author: Greg Waines
:organization: Wind River Systems
:organization: OPNFV - High Availability
:status: Draft
:date: January 2017
:revision: 1.0

:abstract: This document defines a Base Host-to-Guest Messaging Layer 
     that is used to multiplex multiple Application Layer HA Guest APIs 
     on top of a single ‘virtio serial device’ between the Host and the Guest.

.. sectnum::

.. contents:: Table of Contents



Introduction
============

   This document defines a Base Host-to-Guest Messaging Layer that is used to 
   multiplex multiple Application Layer HA Guest APIs on top of a single 
   ‘virtio serial device’ between the Host and the Guest.

   The Base Host-to-Guest Messaging Layer provides a simple low-bandwidth datagram
   messaging service between the VM and the Host.  This messaging channel 
   is available regardless of whether IP networking is functional within 
   the server.  

   This document contains the detailed specification for 
   this messaging-based API and describes the Design of the 
   OpenStack-Host changes in support of this functionality.

   Also included in this document is an overview of a Linux-based 
   reference implementation of the Guest-side software for implementing
   this Base Host-to-Guest Messagign Layer in the Guest.  
   


Host-to-Guest Messaging Layer
=============================

   The Base Host-to-Guest Messaging Layer is a message-based API using a JSON-formatted 
   application messaging layer on top of a ‘virtio serial device’ between QEMU 
   on the OpenStack Host and the Guest VM.  JSON formatting provides a simple, 
   humanly readable messaging format which can be easily parsed and formatted 
   using any high level programming language being used in the Guest VM (e.g. C, 
   Python, Java, etc.).  Use of the ‘virtio serial device’ provides a 
   simple, direct communication channel between host and guest which is
   independent of the Guest’s L2/L3 networking. 

   .. image:: OPNFV_HA_Guest_APIs-Base-Messaging-Layer.png
      :alt: Figure 1 - Base Host-to-Guest Messaging Layer


Virtio Serial Device
--------------------

   The transport layer of the Base Host-to-Guest Messaging Layer API is a
   ‘virtio serial device’ (also known as a ‘vmchannel’) between QEMU (on
   the host) and the Guest VM.  Device emulation in QEMU presents a
   virtio-pci device to the Guest, and a Guest Driver presents a char
   device interface to Guest userspace applications.  This provides a
   simple transport mechanism for communication between the host
   userspace and the guest userspace.  I.e. it is completely independent
   of the networking stack of the Guest, and is available very early in
   the boot sequence of the Guest.
        
   This is a standard Linux QEMU/KVM feature.  The Guest API for
   interfacing with the ‘virtio serial device’ can be found at
   http://www.linux-kvm.org/page/Virtio-serial_API .Generally
   communicating with a ‘virtio serial device’ is very similar to
   communicating via a pipe, or a SOCK_STREAM socket.

   There are however a few additional considerations to be aware of when
   using ‘virtio serial devices’:

   - only one process at a time can open the device in the Guest,
   - read() returns 0, if the Host is not connected to the device,
   - write() blocks or returns -1 with error set to EAGAIN, if the Host
     is not connected,
   - poll() will always set POLLHUP in revents when the Host connection
     is down.  

      + This means that the only way to get event-driven notification of
        connection is to register for SIGIO.  However, then a SIGIO
        event will occur every time the device becomes readable. The
        work-around is to selectively block SIGIO as long as the link is
        up is thought to be up, then unblock it on connection loss so a
        notification occurs when the link comes back.

   - If the Host disconnects the Guest should still process any buffered
     messages from the device,
   - Message boundaries are not preserved, the Guest needs to handle
     message fragment reassembly.  Multiple messages can be returned in
     one read() call, as well as buffers beginning and ending with
     partial messages. This is hard to get perfect; one can study the
     host_guest_msg.c code in the OPNFV High Availability ‘Guest Server
     Group Messaging’ Module for ideas on how this can be handled.

   The QEMU/KVM created by OpenStack in order to host a Guest VM is
   created with a ‘virtio serial device’ named:
   ::

        /dev/virtio-port/cgcs.messaging 
    
   for Base Host–to–Guest VM messaging (i.e. a number of the Application 
   Layer HA Guest APIs will be multiplexed on top of this single virtio 
   serial device via the Base JSON Messaging Layer).
   

JSON Message Layer
------------------

   The upper layer messaging format being used is ‘Line Delimited JSON
   Format’.  I.e. a ‘\n’ character is used to identify message
   boundaries in the stream of data to/from the virtio serial device;
   specifically, a ‘\n’ character is inserted at the start and end of
   the JSON Object representing a Message.
   ::

        \n{key:value,key:value,…}\n

        Note that key and values must NOT contain ‘\n’ characters.



Base JSON Message Layer Syntax
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

   Again, the Base Messaging Layer simply provides the ability to 
   multiplex different groups of message types on top of a single 
   ‘virtio serial device’, e.g. resource scaling versus server group 
   messaging etc.


   **Host – to – Guest Messages**

   :Key: “version”

      :Value: integer
      :Optionality: M
      :Example Value: 1
      :Description: Version of the Base Layer Messaging


   :Key: “source_addr”

      :Value: string
      :Optionality: M
      :Example Value: "cgcs.server_grp"
      :Description: The Host-side addressing of the message;
                    specifically the upper Application Message Layer identifier.


   :Key: “dest_addr”

      :Value: string
      :Optionality: M
      :Example Value: “cgcs.server_grp”
      :Description: The Guest-side addressing of the message;
                    again specifically the upper Application Message Layer identifier.
		    NOTE that the specific Guest being addressed is identified
		    by selecting the appropriate unix socket presented by the Guest's
		    QEMU for communicating with the Guest's virtio-pci device.

   :Key: “data”

      :Value: Line Delimited JSON Formatted String
      :Optionality: M
      :Example Value: "\n{key:value,key:value,…}\n"
      :Description: The Application Layer JSON Message.



   **Guest  – to – Host  Messages**

   Guest – to – Host Messages, from a Base Messaging Layer perspective, are
   identical to Host – to – Guest Messages except for swapped semantics
   of source_addr and dest_addr.



Example
^^^^^^^

   Example of a dummy Application JSON Message Layer Message
   encapsulated inside the Base JSON Messaging Layer.
   ::

     \n{"version":1,"source_addr":"cgcs.dummy”,"dest_addr":"cgcs.dummy”,"data":{"version":1,"msg_type":"dummy_msg_type",:“seq”:1}}\n




Design of OpenStack-Host Base Host-to-Guest Messaging Layer
===========================================================

   This section provides an overview of the design for supporting
   the Base Host-to-Guest Messaging Layer in OpenStack.
        
   Figure 1 in the previous section provides the architecture diagram 
   of the design for supporting the Base Host-to-Guest Messaging Layer 
   in OpenStack, where:

   - Libvirt Patch

      + Checks for one of the new Boolean flavor extraspecs indicating 
        whether the guest supports one of the new HA Guest APIs,
      + If supported, the libvirt changes configure device emulation in
        QEMU to present a virtio-pci device to the VM, for the
        Host-to-Guest communications.

   - Host Agent Process
     which implements the Base JSON Messaging Layer between the Host and
     Guest.
     This includes:

      + opening/reading,/writing and general management of the unix
        socket presented by QEMU for communicating with the Guest over
        the virtio-pci device,
      + parsing/processing/formatting of the Base JSON Messaging Layer
        of the Guest-Host interface, where processing of the messages
        involves:

         * the multiplexing/de-multiplexing of Application Layer
           messages to/from registered Host Application Layer Agents;
           who are responsible for handling the Application Layer -Specific
	   Messaging to/from Guests of the local compute,
         * the interface between the Host Agent Process and 
           these Host Application Layer Agents: 

            - is a message-based interface; 
            - specifically, a JSON Messaging Layer over a UNIX Datagram
              socket containing:

               + the source-addr and dest-addr for the Base JSON
                 Messaging Layer of the Guest-Host Interface, 
               + the instance-address, and
               + an application-level JSON message to be put in the
                 ‘data’ field of the Base JSON Messaging Layer of the
                 Guest-Host Interface.


Reference Design of the Guest Base Host-to-Guest Messaging Layer
================================================================

   This section provides an overview of a Linux-based reference
   implementation of the Guest-side software for implementing this
   Base Host-to-Guest Messaging Layer API in the Guest.
                   
   Figure 1 in the previous section provides the architecture diagram 
   of the reference Guest implementation, where:

   - A Guest Agent Process implements the Base JSON Messaging Layer.
     This includes:

      + opening/reading,/writing and general management of the virtio
        serial device between the Guest and the Host,
      + parsing/processing/formatting of the Base JSON Messaging Layer
        of the Guest-Host interface, where processing of the messages
        involves:

         * the multiplexing/de-multiplexing of Application Layer
           messages to/from registered Guest Application Layer Agents;
           who are responsible for handling the Application Layer -Specific
	   Messages for the Guest,
         * the interface between the Guest Agent Process and the Guest
           Application Process responsible for the specific Application Layer
           in the Guest: 

            - is a message-based interface; 
            - specifically a JSON Messaging Layer over a UNIX Datagram
              socket,

               + where the UNIX Socket Address is the Message Group Type
                 (e.g. cgcs.server_grp for server group messaging) specified
                 within the Base JSON Messaging Layer and 
               + where the JSON Message consists of the ‘data’ field
                 contents specified within the Base JSON Messaging
                 Layer.


