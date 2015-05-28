3  Virtualization Facilities (Host OS, Hypervisor)
====================================================

Requirements:

- The hypervisor should support distrubited HA mechenism so that when hardware or 
  host failure happens, the hypervisor can revocer itself without any instruction
  from the VIM.
- The hypervisor should report its failure and recovery action to the VIM.
- The hypervisor should support VM migration
- The hypervisor should provide isolation for VMs, so that VMs running on the same
  hardware do not impact each other.
- The host OS should provide sufficient process isolation so that VMs running on
  the same hardware do not impact each other.
- Fault detection and reporting capability. The NFVI is responsible for reporting the HW
  and NFVI fault to VIM.
- Hypervisor should detect the failure of the VM. Failure of the VM should be reported to
  the VIM within 1s
- Failure of the hypervisor should be reported to the VIM within 1s
- The hypervisor should record the VM information regularly and provide logs of
  VM actions for future diagnoses.
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
- Process of VIM runing on the compute node should be monitored, and failure of it should
  be notified to the VIM within 1s
