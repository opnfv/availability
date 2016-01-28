6, Multisite Scenario
====================================================

The Multisite scenario refers to the cases when VNFs are deployed on multiple VIMs.
There could be three typical usecases for such scenario.

One is in one DC, multiple openstack clouds are deployed. Taking into consideration that the
number of compute nodes in one openstack cloud are quite limited (nearly 100) for
both opensource and commercial product of openstack, multiple openstack clouds will
have to be deployed in the DC to manage thousands of servers. In such a DC, it should
be possible to deploy VNFs accross openstack clouds.


Another typical usecase is Geographic Redundancy (GR). GR deployment is to deal with more
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


The other usecase is the maintainance. When one site is planning for a maintaining,
it should first replicate the service to another site before it stops them. Such
replication should not disturb the service, nor should it cause any data loss. The
service at the second site should be excuted, before the first site is stopped and
began maintenance. In such case, the multisite schemes may be used.

The multisite scenario is also captured by the Multisite project, in which specific
requirements of openstack are also proposed for different usecases. However,
the multisite project mainly focuses on the requirement of these multisite
usecases on openstack. HA requirements are not necessarily the requirement
for the approaches discussed in multisite. While the HA project tries to
capture the HA requirements in these usecases. The following links are the scenarios
and Usecases discussed in the Multisite project.
https://gerrit.opnfv.org/gerrit/#/c/2123/
https://gerrit.opnfv.org/gerrit/#/c/1438/.


