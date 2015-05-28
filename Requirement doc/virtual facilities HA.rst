3  Virtualization Facilities (Host OS, Hypervisor)
====================================================

Requirements:

- The hypervisor will try to solve HW failure when receiving alarms from the HW. If 
  failed, the hypervisor will inform VIM to take over solving the failure
- The hypervisor should support VM migration
- The hypervisor  should provide isolation for VMs, so that multiple VMs on the same 
  hardware use separate CPU and storage resources. Failure in a certain VM will not 
  influence the service on other VMs.
- The host OS if presents should provide isolation for VMs, so that multiple VMs on the 
  same hardware will not impact each other.
- Fault detection and reporting capability. The NFVI is responsible for reporting the HW 
  and NFVI fault to VIM.
- Hypervisor should detect the failure of the VM. Failure of the VM should be reported to 
  the VIM within 1s 
- Failure of the hypervisor should be reported to the VIM within 1s
- The hypervisor should record VM information regularly and provide the last words of VMs 
  for future diagnoses and analyze.
- The NFVI should maintain the number of VMs provided to the VNF in the face of failures. 
  I.e. the failed VM instances should be replaced by new VM instances
- Large deployments using distributed software-based storage shall separate storage and 
  compute nodes (non-hyperconverged deployment).
- Distributed software-based storage services shall be deployed redundantly.
- Data shall be stored redundantly in distributed storage backends.
- Upon failures of storage services, automatic repair mechanisms (re-build/re-balance of 
  data) shall be triggered automatically.
- Storage data path traffic shall be redundant and fail over within 1 second on link 
  failures.
- Ephemeral storage is not guaranteed to be located on shared storage.
- The storage backend shall support geo-redundancy.
- Notifications about the state of the (distributed) storage backends shall be send to the 
  VIM (in-synch/healthy, re-balancing/re-building, degraded).
..
 [Yifei] Also need vswitch bullet
 [fq] you mean adding requirements about vswitch? I think Ian has already put some 
  contents about the vswitch in the next section.
 [Yifei] It is also needed in this part, maybe they are the same.
- Process of VIM runing on the compute node should be monitored, and failure of it should 
  be notified to the VIM within 1s
..
 [YY] monitor the nova agent, nova agent is running on the compute node. if it fail, we 
  need to notify.
 [MT] recovery VM on the compute node by the nova agent, speed up recovery,
 [Yifei] I really cannot understand what nova agent is. Do you mean nova-compute or all 
  the services provided by nova such as nova-conductor, nova-scheduler, etc?
 As I know in OpenStack, nova-compute is deployed on compute node and others are deployed 
  on control node. That is why I put nova-compute in the compute part below, but I agree 
  that putting it in the hypervisor part is more suitable.
 As I mentioned in the gap doc, the status of nova-compute can be achieved by ServiceGroup 
  API.
 I agree that recovery VM on the compute node to speed up recovery. But I don' t think 
  nova agent has the capability to this work. Here is a link about VM recovery written by 
  Russell Bryant who is the PTL of nova for H & I releases:
    http://blog.russellbryant.net/2015/04/08/implementation-of-pacemaker-managed-openstack-
  vm-recovery/2
    For further details, you can read: https://www.redhat.com/archives/rdo-list/
  2015-April/msg00008.html
 [MT2] Yes, by nova agent I mean what you call the nova-compute. I didn't realize that by 
  nova-compute you mean a process not the host :-)