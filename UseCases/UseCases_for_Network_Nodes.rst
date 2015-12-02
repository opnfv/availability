HA Scenario Analysis
====================

Use cases for network nodes
---------------------------

Neutron-server
^^^^^^^^^^^^^^

It also can be called API server. It is stateless and should be configured as active/active. Currently HAProxy can be used to load-balance requests between all servers. When one of the servers dies, HAProxy can automatically reroute requests away from the dead one.

DHCP agent
^^^^^^^^^^

The DHCP agent can be natively highly available. Neutron has a scheduler which lets you run multiple agents across nodes. You can configure the dhcp_agents_per_network parameter in the neutron.conf file and set it to X (X >=2 for HA, default is 1). All DHCP traffic is broadcast, DHCP servers race to offer IP. All the servers will update the lease tables.

L3 agent
^^^^^^^^

The L3 agent is also natively highly available. The scheduler supports Virtual Router Redundancy Protocol (VRRP) to distribute virtual routers across multiple nodes. To achieve HA, it can be configured in the neutron.conf file.

::
  {

    l3_ha: True

    allow_automatic_l3agent_failover: True

    max_l3_agents_per_router: 2 #or more

    min_l3_agents_per_router: 2 #or more

  }

LBaaS agent
^^^^^^^^^^^

Currently, no native feature is provided to make the LBaaS agent highly available using the default plug-in HAProxy. A common way to make HAProxy highly available is to use Pacemaker. Pacemaker fails over HAProxy and the VIP to another node.

Reference
---------

* OpenStack HA guide  http://docs.openstack.org/ha-guide/networking-ha.html

**Documentation tracking**

Revision: _sha1

Build date:  _date
