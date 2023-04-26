# What is a Crash consistent VM Restore Point? 
A crash consistent VM restore point stores the VM configuration and point-in-time write-order consistent snapshots for all managed disks attached to a Virtual Machine. This is same as the status of data in the VM after a power outage or a crash.

## Get started
You can now create multi-disk crash consistent restore points for your Azure VMs and use these restore points for backup and/or disaster. Crash consistent VM restore points are available in the following regions: EAST US 2 EUAP. If you want know more about VM restore points, please review our [VM restore points public documentation](https://docs.microsoft.com/en-us/azure/virtual-machines/virtual-machines-create-restore-points). Your subscription needs to be whitelisted for using crash consistent restore points so please reach out to VMRestorepoints@microsoft.com to participate in the private preivew.

In preview you can perform the following operation on crash consistent restore points
* Create crash consistent VM restore points to protect Azure VMs at 1 hour frequency 
* Exclude disks that you do not want to protect as part of crash consistent VM restore points to optimize costs
* Restore all disks from a crash consistent VM restore point

All the above operations use the same API interface as app consistent VM restore points. The only changes needed are, when creating a crash consistent VM restore point, you need to specify the "consistencyMode" property as "CrashConsistent" in the request and the request API version should be **2021-07-01** as shown below:

#### URI request
```
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}&api-version={api-version}
```
#### Request body
```
{
    "name": "name of the restore point resource",
    "property":{
        "consistencyMode": "CrashConsistent"
    } 
}
```

## Know issues and limitations in private preview
* Subscription whitelisting is required. Please reachout to VMRestorepoints@microsoft.com with the subscription id to try out this feature.
* Regions supported:CanadaCentral,WestCentralUS,JapanWest, CentralIndia, EastAsia, WestUS. We will continue to rollout in all regions in the coming weeks.
* Suggested frequency at which crash consistent restore points can be created is 1 hour
* Cross region creation of crash consistent restore points directly in a different region than the deployed VM is currently not supported
* Managed Disks of size 8TB and above (striped disks) are currently not supported.
* Managed Disks with paid bursting (striped disks) are not supported.
* Ultra-disks, Premium v2 SSD, Ephemeral OS disks, Shared disks and Write Accelerated disks are not supported.



