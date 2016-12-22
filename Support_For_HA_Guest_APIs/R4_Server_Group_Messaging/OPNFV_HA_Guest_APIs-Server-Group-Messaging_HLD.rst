=========================
Support for HA Guest APIs
=========================
----------------------
Server Group Messaging
----------------------

:author: Greg Waines
:organization: Wind River Systems
:organization: OPNFV - High Availability
:status: Draft
:date: January 2017
:revision: 1.1

:abstract: This document defines a Host-to-Guest Server Group Messaging API and 
     the OpenStack-Host Design to provide a simple low-bandwidth datagram
     messaging and notification service for servers that are part of the
     same server group.  Also included in this document is an overview of
     a Linux-based reference implementation of the Guest-side software for
     implementing this Messaging-based API in the Guest.  

.. sectnum::

.. contents:: Table of Contents



Introduction
============

   This module defines a Host-to-Guest Server Group Messaging API and
   the OpenStack-Host Design to provide a simple low-bandwidth datagram
   messaging and notification service for servers that are part of the
   same server group.  This messaging channel is available regardless of
   whether IP networking is functional within the server, and it 
   requires no knowledge within the server about the other members of 
   the group.  This document contains the detailed specification for 
   this messaging-based API and describes the Design of the 
   OpenStack-Host changes in support of this functionality.

   Also included in this document is an overview of a Linux-based 
   reference implementation of the Guest-side software for implementing
   this Messaging-based API in the Guest.  This Guest-side reference 
   implementation, in this module, provides source code and make/build 
   instructions which can be used strictly as reference or built and 
   included ‘as is’ in your Guest image.  Full build, install and usage
   instructions can be found in the README files included in the 
   Guest-side implementation.  This document simply provides an overview
   of the reference implementation.



Host - Guest Server Group Messaging API
=======================================

   This module implements a simple Host-to-Guest Server Group Messaging
   API to provide a simple low-bandwidth datagram messaging and 
   notification service for servers that are part of the same server
   group.  This messaging channel is available regardless of whether IP
   networking is functional within the server, and it requires no 
   knowledge within the server about the other members of the group.  

   The Host-to-Guest Server Group Messaging API is a message-based API 
   using a JSON-formatted application messaging layer on top of the 
   Base Host-to-Guest Messaging Layer (defined in a peer document
   OPNFV_HA_Guest_APIs-Base-Messaging-Layer.rst).  JSON formatting provides 
   a simple, humanly readable messaging format which can be easily parsed and 
   formatted using any high level programming language being used in the 
   Guest VM (e.g. C, Python, Java, etc.).  


   .. image:: OPNFV_HA_Guest_APIs-Server-Group-Messaging_HLD-FIGURE-1.png
      :alt: Figure 1 - Host-Guest Server Group Messaging API


Message Types and Semantics
---------------------------

   For the Server Group Messaging API, there are four message types; 
   Server Status Query/Response Messages, Asynchronous Server Status 
   Change Notifications, Server Broadcast Messages and a Nack Message.

   - Status Query                                        (Guest -> Host)

     + This allows a server (Guest) to query the current state of all 
       servers within its server group, including itself,

   - Status Response and  Status Response Done           (Host -> Guest)

      + This is the Status Response from the OpenStack Host containing
        the current state of all servers within the Guest’s server 
        group, including this Guest,
      + This is a multiple message response with each response 
        containing the status of a single server, followed by a final 
        response (status response done) with no data,
      + Each message of the multiple message response has the 
        transaction number (seq) of the status query request that it is
        related to,

   - Notification Message                                (Host -> Guest)

      + This asynchronous message provides the server (Guest) with 
        information about changes to the state of other servers within
        the server group,
      + Each notification message contains the status of a single 
        server,

   - Broadcast Message              (Guest -> Host)  ->  (Host -> Guest)

      + This allows a server (Guest) to send a datagram (thru the Host,
        with a size of up to 3050 bytes) to all other servers (Guests) 
        within its server group,

         * the payload portion of the message, ‘data’, can be formatted
           as desired by the Guest, however it must be a null-terminated
           string without embedded newlines,
         * the source field of the message is a unique, although opaque,
           address string representing the server (Guest) that sent the
           message.

   - Nack                                                (Host -> Guest)

      + This is a message sent from the Host to the Guest when the Host
        receives a message with incorrect syntax,
      + It contains the message type of the original (incorrect) message
        and a log_msg describing the error,
      + This allows the Guest Application developer to debug issues when
        developing the Guest-side API code.

   This service is not intended for high bandwidth or low-latency
   operations.  It is best-effort, not reliable.  Applications should
   do end-to-end acks and retries if they care about reliability.



JSON Message Syntax
-------------------

   The upper layer messaging format being used is ‘Line Delimited JSON
   Format’.  I.e. a ‘\n’ character is used to identify message
   boundaries in the stream of data to/from the virtio serial device;
   specifically, a ‘\n’ character is inserted at the start and end of
   the JSON Object representing a Message.
   ::

        \n{key:value,key:value,…}\n

        Note that key and values must NOT contain ‘\n’ characters.



Guest - to - Host Messages
^^^^^^^^^^^^^^^^^^^^^^^^^^

      **Status Query**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface.

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “status_query"          
         :Description: Type of the message.

      :Key: “seq”

         :Value: integer
         :Optionality: M
         :Example Value: 
         :Description: Transaction number for the query;
                       corresponding status_response and
                       status_response_done messages will have a
                       matching transaction number.
                       This should be incremented on each
                       status_query sent by Guest.

      **Broadcast Message**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “broadcast"          
         :Description: Type of the message.

      :Key: “data”

         :Value: string
         :Optionality: M
         :Example Value: 
         :Description: Message content; can be formatted as desired
                       by the Guest, however it must be a
                       null-terminated string without embedded
                       newlines.


Host - to - Guest Messages
^^^^^^^^^^^^^^^^^^^^^^^^^^

      **Status Response**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface.

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “status_response"          
         :Description: Type of the message.

      :Key: “seq”

         :Value: integer
         :Optionality: M
         :Example Value: 
         :Description: Transaction number that the response belongs to.

      :Key: “data”

         :Value: string
         :Optionality: M
         :Example Value: see following info following table
         :Description: The JSON formatted field containing the same
                       contents as the normal notification that gets
                       sent out by OpenStack’s notification service;
                       see example below.

   Example contents of ‘data’ field containing status of a particular
   server:

   ( the same contents as the normal notification that gets sent out by
   OpenStack’s notification service )
   ::

      {  
            "state_description":"",
            "availability_zone":null,
            "terminated_at":"",
            "ephemeral_gb":0,
            "instance_type_id":10,
            "deleted_at":"",
            "reservation_id":"r-ed4i0c72",
            "instance_id":"4c074ce9-cbde-4040-9fdb-84b36168916b",
            "display_name":"jd_af_vm1",
            "hostname":"jd-af-vm1",
            "state":"active",
            "progress":"",
            "launched_at":"2015-11-26T14:33:03.000000",
            "metadata":{  
      
            },
            "node":"compute-0",
            "ramdisk_id":"",
            "access_ip_v6":null,
            "disk_gb":1,
            "access_ip_v4":null,
            "kernel_id":"",
            "host":"compute-0",
            "user_id":"369b0103310d4a6bbf43ed389aac211d",
            "image_ref_url":"http:\/\/127.0.0.1:9292\/images\/32b386e1-5a21-47c4-a04a-57910e7b0fc8",
            "cell_name":"",
            "root_gb":1,
            "tenant_id":"98b5838aa73c40728341336852b07772",
            "created_at":"2015-11-26 14:32:51.431455+00:00",
            "memory_mb":512,
            "instance_type":"jd1cpu",
            "vcpus":1,
            "image_meta":{  
               "min_disk":"1",
               "container_format":"bare",
               "min_ram":"0",
               "disk_format":"qcow2",
               "base_image_ref":"32b386e1-5a21-47c4-a04a-57910e7b0fc8"
            },
            "architecture":null,
            "os_type":null,
            "instance_flavor_id":"101"
      }

   |   

      **Status Response Done**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface.

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “status_response_done"          
         :Description: Type of the message.

      :Key: “seq”

         :Value: integer
         :Optionality: M
         :Example Value: 
         :Description: Transaction number that the response belongs to.



      **Notification Message**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface.

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “notification"          
         :Description: Type of the message.

      :Key: “data”

         :Value: string
         :Optionality: M
         :Example Value: see contents of ‘data’ field documented for
                         status_response 
         :Description: The JSON formatted output of the response to
                       the Compute API GET  /<version>/<tenant_id>/
                       servers/<server_id>

                       ( see Compute API documentation for exact 
                       contents of response )

      **Broadcast Message**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 1
         :Description: Version of the interface

      :Key: “msg_type”

         :Value: string 
         :Optionality: M
         :Example Value: “broadcast"          
         :Description: Type of the message.

      :Key: “source_instance”

         :Value: string
         :Optionality: M
         :Example Value: 
         :Description: The unique, although opaque, address string
                       representing the server (Guest) that sent the
                       message.

      :Key: “data”

         :Value: string
         :Optionality: M
         :Example Value: 
         :Description: Message content; can be formatted as desired
                       by the Guest, however it must be a
                       null-terminated string without embedded
                       newlines.


      **Nack**

      :Key: “version”

         :Value: integer
         :Optionality: M
         :Example Value: 2
         :Description: Version of the interface

      :Key: “msg_type”

         :Value: “nack”
         :Optionality: M
         :Example Value: “nack”
         :Description: The type of message.

      :Key: “orig_msg_type”

         :Value: string
         :Optionality: M
         :Example Value: “broadcast”
         :Description: The type of message that host previous
                       received from guest.

      :Key: “log_msg”

         :Value: string
         :Optionality: M
         :Example Value: “failed to parse version”
         :Description: Error message


Examples
^^^^^^^^

   Examples of ‘full’ Server Group Messaging JSON messages, containing
   the Application JSON Message Layer encapsulated inside the Base 
   Host-to-Guest JSON Messaging Layer (defined in a peer document
   OPNFV_HA_Guest_APIs-Base-Messaging-Layer.rst).


   **Status Query:**
           
   Guest sends a query to OpenStack Host for status of all servers in
   Guest’s Server Group:
   ::

     \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"status_query",:“seq”:1}}\n

   |   

      OpenStack Host responds with the status of a server in the
      Guest’s Server Group; one or more messages, each containing the
      status of one server in the Guest’s Server Group: 
      ::

        \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"status_response",“seq”:1,“data”:{ "state_description": "", "availability_zone": null, "terminated_at": "", "ephemeral_gb": 0, "instance_type_id": 10, "deleted_at": "", "reservation_id": "r-ed4i0c72", "instance_id": "4c074ce9-cbde-4040-9fdb-84b36168916b", "display_name": "jd_af_vm1", "hostname": "jd-af-vm1", "state": "active", "progress": "", "launched_at": "2015-11-26T14:33:03.000000", "metadata": { }, "node": "compute-0", "ramdisk_id": "", "access_ip_v6": null, "disk_gb": 1, "access_ip_v4": null, "kernel_id": "", "host": "compute-0", "user_id": "369b0103310d4a6bbf43ed389aac211d", "image_ref_url": "http:\/\/127.0.0.1:9292\/images\/32b386e1-5a21-47c4-a04a-57910e7b0fc8", "cell_name": "", "root_gb": 1, "tenant_id": "98b5838aa73c40728341336852b07772", "created_at": "2015-11-26 14:32:51.431455+00:00", "memory_mb": 512, "instance_type": "jd1cpu", "vcpus": 1, "image_meta": { "min_disk": "1", "container_format": "bare", "min_ram": "0", "disk_format": "qcow2", "base_image_ref": "32b386e1-5a21-47c4-a04a-57910e7b0fc8" }, "architecture": null, "os_type": null, "instance_flavor_id": "101" }}}\n

      OpenStack Host responds with response done for the current
      outstanding query request; with no data:
      ::

      \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"status_response_done",“seq”:1}}\n

      |   

   **Notification:**

   A notification of a server state change from OpenStack Host:
   ::

     \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"notification",“data”:{ "state_description": "", "availability_zone": null, "terminated_at": "", "ephemeral_gb": 0, "instance_type_id": 10, "deleted_at": "", "reservation_id": "r-ed4i0c72", "instance_id": "4c074ce9-cbde-4040-9fdb-84b36168916b", "display_name": "jd_af_vm1", "hostname": "jd-af-vm1", "state": "active", "progress": "", "launched_at": "2015-11-26T14:33:03.000000", "metadata": { }, "node": "compute-0", "ramdisk_id": "", "access_ip_v6": null, "disk_gb": 1, "access_ip_v4": null, "kernel_id": "", "host": "compute-0", "user_id": "369b0103310d4a6bbf43ed389aac211d", "image_ref_url": "http:\/\/127.0.0.1:9292\/images\/32b386e1-5a21-47c4-a04a-57910e7b0fc8", "cell_name": "", "root_gb": 1, "tenant_id": "98b5838aa73c40728341336852b07772", "created_at": "2015-11-26 14:32:51.431455+00:00", "memory_mb": 512, "instance_type": "jd1cpu", "vcpus": 1, "image_meta": { "min_disk": "1", "container_format": "bare", "min_ram": "0", "disk_format": "qcow2", "base_image_ref": "32b386e1-5a21-47c4-a04a-57910e7b0fc8" }, "architecture": null, "os_type": null, "instance_flavor_id": "101" }}}\n


   **Broadcast:**

   A broadcast message to/from another server:
   ::

     \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"broadcast",“source_instance”:”instance-00000001”,“data”:”Hello World”}}\n


   **Nack:**

   A Nack from OpenStack Host for an invalid broadcast message sent
   from Guest.
   ::

     \n{"version":1,"source_addr":"cgcs.server_grp”,"dest_addr":"cgcs.server_grp”,"data":{"version":1,"msg_type":"nack","orig_msg_type":"broadcast","log_msg":"failed to parse version"}}\n


Design of OpenStack-Host Server Group Messaging 
===============================================

   This section provides an overview of the design for supporting
   Host-to-Guest Server Group Messaging in OpenStack.
        
   The implementation of the OpenStack Host design can be found in the
   OPNFV High Availability ‘Server Group Messaging’ Module.  The design
   attempts to provide a solution that uses OpenStack Core Services, 
   rather than patching the OpenStack Core Services.  The design introduces
   an HA-Guest-Compute process for HA Guest API functionality on the 
   OpenStack Compute Nodes and an HA-Guest-Server process for HA Guest API
   functionality on the OpenStack Controller Nodes.  Interactions with 
   Nova is via public Nova REST APIs and the only required patch is for
   libvirt.  This section provides an overview of the design.
                
   The diagram below provides the architecture diagram of the design
   for supporting Host-to-Guest Server Group Messaging in OpenStack:

   .. image:: OPNFV_HA_Guest_APIs-Server-Group-Messaging_HLD-FIGURE-2.png
      :alt: Figure 2 - Architecture for OpenStack Host support of Guest 
                       Server Group Messaging


   Where:

   - Libvirt Patch

      + see OPNFV_HA_Guest_APIs-Base-Messaging-Layer.rst

   - Host Agent Process

      + see OPNFV_HA_Guest_APIs-Base-Messaging-Layer.rst

   - HA-Guest-Compute - Server Group Messaging MODULE

      + Manages Server Group Messaging on the Compute Node
      + Specifically, it implements the Application JSON Messaging Layer
        for Server Group Messaging on top of the Base JSON Messaging
        Layer UNIX Datagram socket provided by the Host Agent,
      + On receiving a Server Status Query from the Guest

         * HA-Guest-Compute makes an RPC call to HA-Guest-Server to request
           the status of all servers in its server group, and
         * On receiving these back from HA-Guest-Server, forwards them on
           to the Host Agent and the Guest VM,

      + On receiving a Broadcast Message from the Guest

         * HA-Guest-Compute makes an RPC call to HA-Guest-Server to request
           that the Broadcast message be sent to all servers of the
           server group, and 
         * Again on receiving any Broadcast Messages from 
           HA-Guest-Server, forwards them on to the Host Agent and
           therefore the Guest VM. 

   - HA-Guest-Server - Server Group Messaging MODULE

      + Provides centralized functions in support of Server Group
        Messaging
      + Supports an RPC query from HA-Guest-Compute for the status of all
        servers within a server group

         * HA-Guest-Server sends a REST API request to Nova for which 
	   instances are in the server group of the requesting instance, 
	   (potentially caching these for performance reasons)
	 * then sends a REST API request to Nova for the status of all
	   instances that are returned, and
	 * finally sends back a single message containing the status of 
	   all instances to HA-Guest-Compute,

      + Supports an RPC query from HA-Guest-Compute for the broadcasting of
        a message to all servers wthin the requesting instance’s server
        group

         * HA-Guest-Server sends a REST API request to Nova for which 
	   instances are in the server group of the requesting instance, 
	   (again, this info being potentially cached for performance reasons)
	 * then sends a REST API request to Nova to determine which compute nodes 
	   they're on, and 
	   (again, potentially caching and tracking this info for performance reasons)
	 * finally sends one RPC message to each relevant compute node, with a 
	   list of instances to forward to.

      + Hooks into the Nova notification system in order to detect state
        changes in servers and then broadcast that state change
        notification to all servers of the server group; i.e. using the 
	same mechanism as described above for broadcasting to all servers.


Reference Implementation of Guest Server Group Messaging
========================================================

   This section provides an overview of the Linux-based reference
   implementation of the Guest-side software for implementing this
   Host-to-Guest Server Group Messaging API in the Guest.
                   
   This reference implementation can be found in the OPNFV High
   Availability ‘Server Group Messaging’ Module.  This Module provides
   source code and make/build instructions which can be used strictly as
   reference or built and included ‘as is’ in your Guest image.  Full
   build, install and usage instructions can be found in the README
   files included in the module.  This section simply provides an
   overview of the reference implementation.

   The diagram below provides the architecture diagram of the reference
   implementation:

   .. image:: OPNFV_HA_Guest_APIs-Server-Group-Messaging_HLD-FIGURE-3.png
      :alt: Figure 3 - Reference Implementation Architecture for  
                       Guest Server Group Messaging


   Where:

   - A Guest Agent Process implements the Base JSON Messaging Layer.

      + see OPNFV_HA_Guest_APIs-Base-Messaging-Layer.rst

   - A Server Group Messaging Lib which provides a C-based procedural
     API for a Guest Application Process to interface with the Guest
     Agent Process.

     Specifically this library implements:

      + the interface described above; a JSON Messaging Layer over a
        UNIX Datagram socket.  

         * where the UNIX Socket Address is the Message Group Type
           (cgcs.server_grp in this particular case) specified within
           the Base JSON Messaging Layer and 
         * the JSON Message consists of the ‘data’ field contents
           specified within the Base JSON Messaging Layer.

      + with a C-based procedural API. 

      + NOTE

         * the definition and implementation of the Server Group
           Messaging Lib within the OPNFV High Availability ‘Server
           Group Messaging’ Module are:

            - server_group.h and server_group.c
            - server_group_app.c   (a sample usage of the API)


server_group.h::

   /* Function signature for the server group broadcast messaging callback
    * function.  source_instance is a null-terminated string of the form
    * "instance-xxxxxxxx".  The message contents are entirely up to the
    *  sender of the message.
    */
   typedef void (*sg_broadcast_msg_handler_t)(const char *source_instance,
                 const char *msg, unsigned short msglen);
   
   /* Function signature for the server group notification callback 
    * function.  The message is basically the notification as sent out by
    * nova with some information removed as not relevant.  The message is
    * not null-terminated, though it is a JSON representation of a python
    * dictionary.
    */
   typedef void (*sg_notification_msg_handler_t)(const char *msg, 
                 unsigned short msglen);
   
   /* Function signature for the server group status callback function.  
    * The message is a JSON representation of a list of dictionaries, 
    * each of which corresponds to a single server.  The message is not 
    * null-terminated.
    */
   typedef void (*sg_status_msg_handler_t)(const char *msg, 
                 unsigned short msglen);
   
   
   
   
   /* Get error message from most recent library call that returned an
    * error. 
    */
   char *sg_get_error();
   
   /* Allocate socket, set up callbacks, etc.  This must be called once
    * before any other API calls.
    *
    * Returns a socket that must be monitored for activity using 
    * select/poll/etc.
    * A negative return value indicates an error of some kind.
    */
   int init_sg(sg_broadcast_msg_handler_t broadcast_handler,
               sg_notification_msg_handler_t notification_handler,
               sg_status_msg_handler_t status_handler);
   
   /* This should be called when the socket becomes readable.  This may
    * result in callbacks being called.  Returns 0 on success.
    * A negative return value indicates an error of some kind.
    */
   int process_sg_msg();
   
   /* max msg length for a broadcast message */
   #define MAX_MSG_DATA_LEN 3050
   
   
   
   /* Send a server group broadcast message.  Returns 0 on success.
    * A negative return value indicates an error of some kind.
    */
   int sg_msg_broadcast(const char *msg);
   
   /* Request a status update for all servers in the group.
    * Returns 0 if the request was successfully sent.
    * A negative return value indicates an error of some kind.
    *
    * A successful response will cause the status_handler callback
    * to be called.
    *
    * If a status update has been requested but the callback has not yet
    * been called this may result in the previous request being cancelled.
    */
   int sg_request_status();

