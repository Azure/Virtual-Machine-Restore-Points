# Cross-Region-VM-Restore-Points
As an extension to VM Restore Points we are providing additional functionality within Azure platform to enable our partners to build BCDR solutions for Azure VMs. One such functionality is: 
Ability to copy VM Restore Points from one region to another other region
 
 Scenarios where this API can be helpful:
 * Extend multiple copies of backup to different regions
 * Extend local backup solutions to support disaster recovery from region failures

**NOTE 1:** For copying a RestorePoint across region, you need to pre-create a RestorePoint in the local region. For details regarding creation of VM Restore Points in the local region refer to the [VM Restore Points documentation](https://github.com/Azure/Virtual-Machine-Restore-Points).

**NOTE 2:** Cross Region Copy of VM Restore Point feature is currently in private preview and is not meant for production workloads. The feature is currently supported via REST APIs only. Other client tool support such as portal, CLI, SDKs, etc. will be coming later. 

**NOTE 3:** Cross Region Copy of VM Restore Point feature is supported starting with api-version: '2021-03-01'.

## Cross-region copy of VM Restore Points
### Create Restore Point Collection in target region
First step in copying an existing VM Restore point from one region to another is to create a RestorePointCollection in the target region by referencing the RestorePointCollection from the source region.

#### URI Request
```
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}&api-version={api-version}
```

#### Request Body
```
{
    "name": "name of the copy of restorePointCollection resource",
    "location": "location of the copy of the restorePointCollection resource",    
    "tags": {
        "department": "finance"
    },
    "properties": {
         "source": {
               "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/restorePointCollections/{restorePointCollectionName}"
         }
    }
}
```

#### Response
The request response will include a status code and set of response headers.
##### Status code
The operation returns a 201 during create and 200 during Update.
##### Response body
```
{
    "name": "name of the copied restorePointCollection resource",
    "id": "CSM Id of copied restorePointCollection resource",
    "type": "Microsoft.Compute/restorePointCollections",
    "location": "location of the copied restorePointCollection resource",
    "tags": {
        "department": "finance"
    },
    "properties": {
        "source": {
            "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/restorePointCollections/{restorePointCollectionName}",
            "location": "location of source RPC"
        }
    }
}
```

### Create VM Restore Point in Target Region
Next step is to trigger creation of a RestorePoint in the target RestorePointCollection referencing the RestorePoint in the source region that needs to be copied.

#### URI request
```
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}&api-version={api-version}
```
#### Request body
```
{
    "name": "name of the restore point resource",
    "sourceRestorePoint": {
        "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}"
    }
}
```

**NOTE:** Location of the sourceRestorePoint would be inferred from that of the source RestorePointCollection

#### Response
The request response will include a status code and set of response headers. 

##### Status Code
This is a long running operation; hence the operation returns a 201 during create. The client is expected to poll for the status using the operation. (Both the Location and Azure-AsyncOperation headers are provided for this purpose.) 

During restore point creation, the ProvisioningState would appear as Creating in GET restore point API response. If creation fails, its ProvisioningState will be Failed. ProvisioningState would be set to Succeeded when the data copy across regions is initiated. 

**NOTE:** You can track the copy status by calling GET instance View (?$expand=instanceView) on the target VM Restore Point. Please check the "Get VM Restore Points Copy/Replication Status" section below on how to do this. VM Restore Point is considered usable (can be used to restore a VM) only when copy of all the disk restore points are successful.

##### Response body
```
{
    "id": "CSM Id of the restore point",
    "name": "name of the restore point",
    "optionalProperties": "opaque bag of properties to be passed to extension",
    "sourceRestorePoint": {
        "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}"
    },
    "consistencyMode": "CrashConsistent | FileSystemConsistent | ApplicationConsistent",
    "sourceMetadata": {
        "vmId": "Unique Guid of the VM from which the restore point was created",
        "location": "source VM location",
        "hardwareProfile": {
            "vmSize": "Standard_A1"
        },
        "osProfile": {
            "computername": "",
            "adminUsername": "",
            "secrets": [
                {
                    "sourceVault": {
                        "id": "/subscriptions/<subId>/resourceGroups/<rgName>/providers/Microsoft.KeyVault/vaults/<keyvault-name>"
                    },
                    "vaultCertificates": [
                        {
                            "certificateUrl": "https://<keyvault-name>.vault.azure.net/secrets/<secret-name>/<secret-version>",
                            "certificateStore": "certificateStoreName on Windows"
                        }
                    ]
                }
            ],
            "customData": "",
            "windowsConfiguration": {
                "provisionVMAgent": "true|false",
                "winRM": {
                    "listeners": [
                        {
                            "protocol": "http"
                        },
                        {
                            "protocol": "https",
                            "certificateUrl": ""
                        }
                    ]
                },
                "additionalUnattendContent": [
                    {
                        "pass": "oobesystem",
                        "component": "Microsoft-Windows-Shell-Setup",
                        "settingName": "FirstLogonCommands|AutoLogon",
                        "content": "<XML unattend content>"
                    }
                ],
                "enableAutomaticUpdates": "true|false"
            },
            "linuxConfiguration": {
                "disablePasswordAuthentication": "true|false",
                "ssh": {
                    "publicKeys": [
                        {
                            "path": "Path-Where-To-Place-Public-Key-On-VM",
                            "keyData": "PEM-Encoded-public-key-file"
                        }
                    ]
                }
            }
        },
        "storageProfile": {
            "osDisk": {
                "osType": "Windows|Linux",
                "name": "OSDiskName",
                "diskSizeGB": "10",
                "caching": "ReadWrite",
                "managedDisk": {
                    "id": "CSM Id of the managed disk",
                    "storageAccountType": "Standard_LRS"
                },
                "diskRestorePoint": {
                    "id": "/subscriptions/<subId>/resourceGroups/<rgName>/restorePointCollections/<rpcName>/restorePoints/<rpName>/diskRestorePoints/<diskRestorePointName>"
                }
            },
            "dataDisks": [
                {
                    "lun": "0",
                    "name": "datadisk0",
                    "diskSizeGB": "10",
                    "caching": "ReadWrite",
                    "managedDisk": {
                        "id": "CSM Id of the managed disk",
                        "storageAccountType": "Standard_LRS"
                    },
                    "diskRestorePoint": {
                        "id": "/subscriptions/<subId>/resourceGroups/<rgName>/restorePointCollections/<rpcName>/restorePoints/<rpName>/diskRestorePoints/<diskRestorePointName>"
                    }
                }
            ]
        },
        "diagnosticsProfile": {
            "bootDiagnostics": {
                "enabled": true,
                "storageUri": " http://storageaccount.blob.core.windows.net/"
            }
        }
    },
    "provisioningState": "Succeeded | Failed | Creating | Deleting",
    "provisioningDetails": {
        "creationTime": "Creation Time of Restore point in UTC"
    }
}
```

### Get VM Restore Points Copy/Replication Status
Once copy of VM Restore Points is initiated, you can track the copy status by calling GET instance View (?$expand=instanceView) on the target VM Restore Point.

#### URI Request
```
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?$expand=instanceView&api-version={api-version}
```

#### Response
```
{
    "id": "CSM Id of the restore point",
    "name": "name of the restore point",
    "optionalProperties": "opaque bag of properties to be passed to extension",
    "sourceRestorePoint": {
        "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}"
    },
    "consistencyMode": "CrashConsistent | FileSystemConsistent | ApplicationConsistent",
    "sourceMetadata": {
        "vmId": "Unique Guid of the VM from which the restore point was created",
        "location": "source VM location",
        "hardwareProfile": {
            "vmSize": "Standard_A1"
        },
        "osProfile": {
            "computername": "",
            "adminUsername": "",
            "secrets": [
                {
                    "sourceVault": {
                        "id": "/subscriptions/<subId>/resourceGroups/<rgName>/providers/Microsoft.KeyVault/vaults/<keyvault-name>"
                    },
                    "vaultCertificates": [
                        {
                            "certificateUrl": "https://<keyvault-name>.vault.azure.net/secrets/<secret-name>/<secret-version>",
                            "certificateStore": "certificateStoreName on Windows"
                        }
                    ]
                }
            ],
            "customData": "",
            "windowsConfiguration": {
                "provisionVMAgent": "true|false",
                "winRM": {
                    "listeners": [
                        {
                            "protocol": "http"
                        },
                        {
                            "protocol": "https",
                            "certificateUrl": ""
                        }
                    ]
                },
                "additionalUnattendContent": [
                    {
                        "pass": "oobesystem",
                        "component": "Microsoft-Windows-Shell-Setup",
                        "settingName": "FirstLogonCommands|AutoLogon",
                        "content": "<XML unattend content>"
                    }
                ],
                "enableAutomaticUpdates": "true|false"
            },
            "linuxConfiguration": {
                "disablePasswordAuthentication": "true|false",
                "ssh": {
                    "publicKeys": [
                        {
                            "path": "Path-Where-To-Place-Public-Key-On-VM",
                            "keyData": "PEM-Encoded-public-key-file"
                        }
                    ]
                }
            }
        },
        "storageProfile": {
            "osDisk": {
                "osType": "Windows|Linux",
                "name": "OSDiskName",
                "diskSizeGB": "10",
                "caching": "ReadWrite",
                "managedDisk": {
                    "id": "CSM Id of the managed disk",
                    "storageAccountType": "Standard_LRS"
                },
                "diskRestorePoint": {
                    "id": "/subscriptions/<subId>/resourceGroups/<rgName>/restorePointCollections/<rpcName>/restorePoints/<rpName>/diskRestorePoints/<diskRestorePointName>",
                }
            },
            "dataDisks": [
                {
                    "lun": "0",
                    "name": "datadisk0",
                    "diskSizeGB": "10",
                    "caching": "ReadWrite",
                    "managedDisk": {
                        "id": "CSM Id of the managed disk",
                        "storageAccountType": "Standard_LRS"
                    },
                    "diskRestorePoint": {
                        "id": "/subscriptions/<subId>/resourceGroups/<rgName>/restorePointCollections/<rpcName>/restorePoints/<rpName>/diskRestorePoints/<diskRestorePointName>",
                    }
                }
            ]
        },
        "diagnosticsProfile": {
            "bootDiagnostics": {
                "enabled": true,
                "storageUri": " http://storageaccount.blob.core.windows.net/"
            }
        },
    },
    "provisioningState": "Succeeded | Failed | Creating | Deleting",
    "provisioningDetails": {
        "creationTime": "Creation Time of Restore point in UTC"
    },
    "instanceView": {
        "statuses": [
            {
                "code": "ReplicationState/succeeded",
                "level": "Info",
                "displayStatus": "Replication succeeded",
            }
        ],
        "diskRestorePoints": [
            {
                "id": "<diskRestorePoint Arm Id>",
                "replicationStatus": {
                    "status": {
                        "code": "ReplicationState/succeeded",
                        "level": "Info",
                        "displayStatus": "Replication succeeded",
                    },
                    "completionPercent": "<completion percentage of the replication>"
                }
            }
        ]
    }
}
```

## Known issues and limitations
* Cross-region copy functionality is only supported via REST APIs. ARM Template deployment, CLI, SDK, PS will be added later. 
* Copy of copy does not work - You cannot copy a restore point that is already copied from another region. For ex. if you have copied a RP1 from East US to West US as RRP1. You cannot copy RRP1 from West US to another region (or back to East US). 
* A single Restore Point in the source region can only be copied once to a target region. Meaning you cannot have multiple copies of the same restore point in a single target region.
* Copy of the Restore point of CMK encrypted VM and disks will be encrypted using PMK in the target region. Customer needs to use CMK when restoring the disks and VM from the restore point.
* Target Restore Point only shows the creation time when the target Restore Point was created. To get the creation time of the original restore point, you need to call GET on the original restore point. 
* Currently, the replication progress in only updated once every 10mins. Hence for disks that have low churn, there can be scenarios where only the initial (0) and the terminal replication progress (100) can be seen.
* Maximum copy time that is supported is 2 weeks. If there is huge amount data to be copied to the target region, depending on the bandwidth available between the regions, the copy time could be couple of days. If the copy time exceeds 2 weeks, the copy operation will be terminated. 
* There are no error details provided when a Disk Restore Point copy fails
* When a disk restore point copy fails,  intermediate completion percentage where the copy failed is not shown.
* Restoring of Disk from Restore point does not automatically check if the disk restore points replication is completed. You need to manually check the replication status and start restoring the disk only after the disk restore points is replicated successfully.
* Out of order copying of Restore Points is not supported. Meaning, if you have 3 restore points - RP1, RP2, RP3. If you have copied RP1 and RP3 to target region, you cannot copy RP2. 
* Currently, the restore points and cross region copy charges get billed on an incorrect/non-existent resource. On clicking the resource it shows that it doesn't exist/not found.

## Frequently asked questions (FAQs)
**Question: Can I copy the same RPC and RPs to multiple regions (one-to-many) independently to one another? Can this be done in parallel?**
Answer: Yes

**Question: On the VM Restore Points Copy/Replication Status: How can I get the original RP creation time? Is it provisioningDetails.creationTime?**
Answer: GET on the Restore Point will return the creation time of that Restore Point. To get the creation time of the original Restore Point, you need to call GET on the original Restore Point.   

**Question: Can I start work on a specific Disk Restore Point once it is copied to the target region (replicationStatus.Status.Code=succeeded) even if other disk restore points copy has NOT completed yet or failed?**
Answer: Yes. However, Restoring of Disk from Restore point does not automatically check if the disk restore points replication is completed. You need to manually check the replication status and start restoring the disk only after the disk restore points is replicated successfully.

**Question: Does the copy API support Encrypted VMs and disks that use SSE-CMK? What is the behavior in the target region? Will the target VM and disks be encrypted with SSE-CMK?**
Answer: The disk restore points copied to the target region are encrypted using PMK. While restoring disks from these disk restore points you need to provide the CMK for encrypting the restore disk using SSE-CMK.    

**Question: Is copy of restore point within the same region supported?**
Answer: No

**Question: Are there any limitation to the number of restore points that can be copied?**
Answer: Each VM can have a maximum of 200 Restore Points. So you can only copy 200 Restore Points from the source Retore Point Collection to a target Restore Point Collection.

**Question: Can I copy multiple restore points from a source to a target region parallelly?**
Answer: No. Copying of another restore point from the same source to the same target is allowed only after the previous copy is complete.

**Question: Can I copy restore points from a source to a target region out-of-order?**
Answer: No

**Question: Can I copy of Restore Points from different source Restore Point Collections to the same Restore Point Collection in the target region?**
Answer: No

**Question: Can I exclude disks while copying restore points from source region to target region?**
Answer: No, you need to exclude disks while creating the local region restore point.

**Question: ProvisioningState of RestorePoint is marked as succeeded, does that mean copy of RestorePoint is completed/successful?**
Answer: No, provisioningState only indicates that copy has been initiated successfully.  Poll on ReplicationState via Get instanceView for actual copy status