# What is a VM Restore Point? 

A VM Restore Point stores the VM configuration and point-in-time crash (if the VM is shutdown) or application consistent snapshot for all managed disks attached to a Virtual Machine. VMRestorePointCollection is the ARM resource which contains the restore points specific to a VM and each Restore Point contains disk restore points for each included disk. The resource hierarchy hence looks like this-


   VM Restore Points Collection

           VM Restore Points (application or crash consistent across disks at a point in time)
    
                 Disk Restore Points (one for each disk included in the VM restore point)

You can create a VM using the VM Restore Point or create individual disks from the Disk Restore Point object. VM Restore Points are incremental where the first Restore Point stores a full copy of all the disk attached to the VM. For each successive restore point for a VM, only the incremental changes to your disks are backed up. To further reduce your costs, you can optionally exclude any disk when creating a restore point for your VM. 

## Note
The Vm Restore Point feature is currently in private preview and is not meant for production workloads. The feature is currently supported via ARM templates and REST APIs only. Other client tool support such as portal, CLI, SDKs, etc. will be coming later. 

## Restrictions
1. Only works with Managed disks.
2. Ultra disks, Ephemeral OS Disks and Shared Disks are not supported.
3. Requires API version >= 2020-06-01

## Creating a VM Restore Point
1. First Step is to create a RestorePointCollection. You can do so by using the template https://github.com/Azure/Virtual-Machine-Restore-Points/blob/main/createRestorePointCollection.json
2. Next, you can create a restore point using the RestorePointCollection. You can do so by using the template https://github.com/Azure/Virtual-Machine-Restore-Points/blob/main/CreateRestorePoint.json

2a. You can also create a restore point and exclude one or more attached disks by using the following API call:

PUT https://management.azure.com/subscriptions/{subscriptionID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{RestorePointCollectionName}/restorePoints/{RestorePointName}?api-version=2020-06-01

Request Body:
{
"name": "<RestorePointName>",
"excludeDisks": [ {"id": "{/subscriptions/{subscriptionID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.Compute/disks/{diskName}}"}],
"id" : "apientityref"
}

## RestorePointCollection Resource
Use the following URI for GET and DELETE operation on the Restore Point Collection resource. The URI has all the required parameters and there is no need for an additional request body.

https://management.azure.com/subscriptions/{SubscriptionID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{RestorePointCollectionName}?$expand=restorePoints&api-version=2020-06-01
 
You can use PATCH/PUT request to update tags on a Restore Point Collection. No other properties (e.g. location, source VM) can be updated. 

## RestorePoint Resource
Use the following URI for GET and DELETE operation on the Restore Point resource. The URI has all the required parameters and there is no need for an additional request body.

https://management.azure.com/subscriptions/{SubscriptionID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{RestorePointCollectionName}/RestorePoints/{RestorePointName}?api-version=2020-06-01
 
An Update on existing Restore Point Collection resource is not supported. 

## Create Disk

1. Get the Disk Restore ID by either using GET call on Restore Point Collection and expanding Restore Points or by doing a call on Restore Point(s). 
2. You can then use this ARM template to create a disk using diskRestorePoint: https://github.com/Azure/Virtual-Machine-Restore-Points/blob/main/createDiskFromDiskRestorePoint.json

Alternatively, you can also use the below Rest API call (same as https://docs.microsoft.com/en-us/rest/api/compute/disks/createorupdate):

PUT https://management.azure.com/subscriptions/49151e6d-fa6e-4985-ae85-00548ec78853/resourceGroups/dfgdf_group/providers/Microsoft.Compute/disks/restore_disk1?api-version=2020-12-01

Request Body:
{
"location": "East US",
"properties": {
"creationData": {
"createOption": "Restore",
"sourceResourceId": "/subscriptions/{SubscriptionID}/resourceGroups/dfgdf_group/providers/Microsoft.Compute/restorePointCollections/{RestorePointCollectionName}/restorePoints/{RestorePointName}/diskRestorePoints/{DiskRestorePointName}"
}
}
}

## SAS using Disk Restore Points

We are providing a BeginGetAccess API through which the user can directly pass the ID of the diskRestorePoint to create a SAS to the underlying disk. 

   If there is no active SAS on the incremental snapshot of the restore point, then a new SAS will be created and Url returned to the user. 

   If there is already an active SAS, the SAS will be extended and original SASUrl will be returned. 

POST  

https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Compute/restorepointcollections/{restorePointCollection}/restorepoints/{vmRestorePoint}/diskRestorePoints/{diskRestorePointName}/BeginGetAccess

POST request body  

{  

    "access" : "Read"  

    "durationInSeconds": "3600"  

    "revokeReadAccessIfExists" : true  

}  
Response codes  

202 Accepted. This operation is performed asynchronously. After receiving a 202 HTTP response, the client is expected to poll for the status of the asynchronous part of the operation using the URL returned in the Azure-AsyncOperation header. Azure will show operation status as complete (‘succeeded’) only after the operation is complete.  

There is also a EndGetAccess API in which the user can directly pass the ID of the diskRestorePoint to revoke SAS.

POST  

https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Compute/restorepointcollections/{restorePointCollection}/restorepoints/{vmRestorePoint}/diskRestorePoints/{diskRestorePointName}/EndGetAccess  

Response codes  

202 Accepted. This operation is performed asynchronously. After receiving a 202 HTTP response, the client is expected to poll for the status of the asynchronous part of the operation using the URL returned in the Azure-AsyncOperation header. Azure will show operation status as complete (‘succeeded’) only after the operation is complete.  

