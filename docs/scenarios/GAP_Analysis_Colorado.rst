Introduction: 
^^^^^^^^^^^^^

During the Colorado release the OPNFV availability team has reviewed a number of gaps
in support for high availability in various areas of OPNFV.  The focus and goal was
to find gaps and work with the various open source communities( OpenStack as an
example ) to develop solutions and blueprints.  This would enhance the overall
system availability and reliability of OPNFV going forward.  We also worked with
the OPNFV Doctor team to ensure our activities were coordinated.  In the next
releases of OPNFV the availability team will update the status of open gaps and
continue to look for additional gaps.

Summary of findings:
^^^^^^^^^^^^^^^^^^^^

1. Publish health status of compute node - this gap is now closed through and
OpenStack blueprint in Mitaka

2. Health status of compute node - some good work underway in OpenStack and with
the Doctor team, we will continue to monitor this work.

3. Store consoleauth tokens to the database - this gap can be address through
changing OpenStack configurations

4. Active/Active HA of cinder-volume - active work underway in Newton, we will
monitor closely

5. Cinder volume multi-attachment - this work has been completed in OpenStack -
this gap is now closed

6. Add HA tests into Fuel - the Availability team has been working with the
Yardstick team to create additional test case for the Colorado release.  Some of
these test cases would be good additions to installers like Fuel.

Detailed explanation of the gaps and findings:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

GAP 1: Publish the health status of compute node
================================================

* Type: 'reliability'
* Description:

   Current compute node status is only kept within nova. However, NFVO and VNFM
   may also need these information. For example, NFVO may trigger scale up/down
   based on the status. VNFM may trigger evacuation. In the meantime, in the
   high availability scenarios, VNFM may need the host status info from the VIM
   so that it can figure out what the failure exactly located. Therefore, these
   info need to be published outside to the NFVO and VNFM.

 + Desired state

   - Be able to have the health status of compute nodes published.

 + Current behaviour
 
   - Nova queries the ServiceGroup API to get the node liveness information.

 + Gap

- Currently Service Group is keeping the health status of compute nodes internal
- within nova, could have had those status published to NFV MANO plane.

Findings:

BP from the OPNFV Doctor team has covered this GAP. Add notification for service
status change.

Status: Merged (Mitaka release)

 + Owner: Balazs

 + BP: https://blueprints.launchpad.net/nova/+spec/service-status-notification

 + Spec: https://review.openstack.org/182350

 + Code: https://review.openstack.org/#/c/245678/

 + Merged Jan 2016 - Mitaka

GAP 2: Health status of compute node
====================================

* Type: 'reliability'
* Description:

 + Desired state:

   - Provide the health status of compute nodes.

 + Current Behaviour

   - Currently , while performing some actions like evacuation, Nova is checking for the compute service. If the service is down,it is assumed the host is down. This is not exactly true, since there is a possibility to only have compute service down, while all VMs that are running on the host, are actually up. There is no way to distinguish between two really different things: host status and nova-compute status, which is deployed on the host.
   - Also, provided host information by API and commands, are service centric, i.e."nova host-list" is just another wrapper for "nova service-list" with different format (in fact "service-list" is a super set to "host-list").
 

 + Gap

   - Not all the health information of compute nodes can be provided. Seems like nova is treating *host* term equally to *compute-host*, which might be misleading. Such situations can be error prone for the case where there is a need to perform host evacuation.


Related BP:

Pacemaker and Corosync can provide info about the host. Therefore, there is
requirement to have nova support the pacemaker service group driver. There could
be another option by adding tooz servicegroup driver to nova, and then have to
support corosync driver.

  + https://blueprints.launchpad.net/nova/+spec/tooz-for-service-groups

Doctor team is not working on this blueprint

NOTE: This bp is active. A suggestion is to adopt this bp and add a corosync
driver to tooz. Could be a solution.

We should keep following this bp, when it finished, see if we could add a
corosync driver for tooz to close this gap.

Here are the currently supported driver in tooz.
https://github.com/openstack/tooz/blob/master/doc/source/drivers.rst Meanwhile,
we should also look into the doctor project and see if this could be solved.

This work is still underway, but, doesn't directly map to the gap that it is
identified above.  Doctor team looking to get faster updates on node status and
failure status - these are other blueprints.  These are good problems to solve.

GAP 3: Store consoleauth tokens to the database
===============================================

* Type: 'performance'
* Description:

+ Desired state

   - Change the consoleauth service to store the tokens in the databaseand, optionally, cache them in memory as it does now for fast access.

+ Current State

   - Currently the consoleauth service is storing the tokens and theconnection data only in memory. This behavior makes impossible to have multipleinstances of this service in a cluster as there is no way for one of theisntances to know the tokens issued by the other.

   - The consoleauth service can use a memcached server to store those tokens,but again, if we want to share them among different instances of it we would berelying in one memcached server which makes this solution unsuitable for a highly available architecture where we should be able to replicate all ofthe services in our cluster.

+ Gap

   - The consoleauth service is storing the tokens and the connection data only in memory. This behavior makes impossible to have multiple instances of this service in a cluster as there is no way for one of the instances to know the tokens issued by the other.

* Related BP

 + https://blueprints.launchpad.net/nova/+spec/consoleauth-tokens-in-db

 The advise in the blueprint is to use memcached as a backend. Looking to the
 documentation memcached is not able to replicate data, so this is not a
 complete solution. But maybe redis (http://redis.io/) is a suitable backend
 to store the tokens that survive node failures.  This blueprint is not
 directly needed for this gap.

Findings:

This bp has been rejected since the community feedback is that A/A can be 
supported by memcacheD. The usecase for this bp is not quite clear, since when 
the consoleauth service is done and the token is lost, the other service can 
retrieve the token again after it recovers.  Can be accomplished through a 
different configuration set up for OpenStack.  Therefore not a gap.  
Recommendation of the team is to verify the redis approach.


GAP 4: Active/Active HA of cinder-volume
========================================

* Type: 'reliability/scalability' 

* Description:

 + Desired State:

   - Cinder-volume can run in an active/active configuration.

 + Current State:

   - Only one cinder-volume instance can be active. Failover to be handledby external mechanism such as pacemaker/corosync.

 + Gap

   - Cinder-volume doesn't supprt active/active configuration.

* Related BP

  + https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

* Findings:

  + This blueprint underway for Newton - as of July 6, 2016 great progress has been made, we will continue to monitor the progress.  

GAP 5: Cinder volume multi-attachment
=====================================

* Type: 'reliability'
* Description:

 + Desired State

   - Cinder volumes can be attached to multiple VMs at the same time.  So that active/standby stateful VNFs can share the same Cinder volume.

 + Current State
 
   - Cinder volumes can only be attached to one VM at a time.
 
 + Gap
 
   - Nova and cinder do not allow for multiple simultaneous attachments.
 
* Related BP
 
  + https://blueprints.launchpad.net/openstack/?searchtext=multi-attach-volume

* Findings

  + Multi-attach volume is still WIP in OpenStack.  There is coordination work required with Nova.
  + At risk for Newton
  + Recommend adding a Yardstick test case.

General comment for the next release.  Remote volume replication is another important project for storage HA.
The HA team will monitor this multi-blueprint activity that will span multiple OpenStack releases.  The
blueprints aren't approved yet and there dependencies on generic-volume-group.



GAP 6: HA tests improvements in fuel
====================================

* Type: 'robustness'
* Description:

  + Desired State
    - Increased test coverage for HA during install
  + Current State
    - A few test cases are available

  * Related BP

    - https://blueprints.launchpad.net/fuel/+spec/ha-test-improvements
    - Tie in with the test plans we have discussed previously.
    - Look at Yardstick tests that could be proposed back to Openstack.
    - Discussions planned with Yardstick team to engage with Openstack community to enhance Fuel or Tempest as appropriate.


Next Steps:
^^^^^^^^^^^

The six gaps above demonstrate that on going progress is being made in various
OPNFV and OpenStack communities.  The OPNFV-HA team will work to suggest
blueprints for the next OpenStack Summit to help continue the progress of high
availability in the community.
