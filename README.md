# What is a VM Restore Point? 
A VM Restore Point stores the VM configuration and point-in-time crash (if the VM is shutdown) or application consistent snapshot for all managed disks attached to a Virtual Machine. VMRestorePointCollection is the ARM resource which contains the restore points specific to a VM and each Restore Point contains disk restore points for each included disk. The resource hierarchy hence looks like this-

   VM Restore Points Collection

           VM Restore Points (application consistent across disks at a point in time)
    
                 Disk Restore Points (one for each disk included in the VM restore point)

You can create a VM using the VM Restore Point or create individual disks from the Disk Restore Point object. VM Restore Points are incremental where the first Restore Point stores a full copy of all the disk attached to the VM. For each successive restore point for a VM, only the incremental changes to your disks are backed up. To further reduce your costs, you can optionally exclude any disk when creating a restore point for your VM. 

## Get started
VM restore points are available in all public regions. Please review the [VM restore points documentation]() to learn how to:
* Create VM restore points to protect the VM from data loss and data corruption 
* Exclude disks that you do not want to protect as part of VM restore points to optimize costs
* Restore a VM and all disks from a VM restore point

## Cross Region VM Restore Points (Preview)
As an extension to VM Restore Points we are providing additional functionality within Azure platform to enable our partners to build BCDR solutions for Azure VMs. Additional functionalities include: 
**1. Ability to copy VM Restore Points from one region to another other region**
 Scenarios where this API can be helpful:
 * Extend multiple copies of backup to different regions
 * Extend local backup solutions to support disaster recovery from region failures

**2. Ability to create VM Restore Points directly in the target region by referencing a VM in the source region**
Scenario where this API can be helpful: 
* Implement a disaster recovery solution to protect VMs from region failure.

NOTE: Currently cross region VM restore points functionality is in private preview. If you are intersted in testing these capabilities please reach out to your Microsoft account executive.

