### ```NOTE: Cross region creation of VM Restore points has been deperacted```
## Cross-region creation of VM Restore Points
Ability to create VM Restore Points directly in the target region by referencing a VM in the source region
**Scenario where this API can be helpful:** Implement a disaster recovery solution to protect VMs from region failure.

### Create Restore Point Collection in target region
First step in creating a VM Restore point in a target region referencing a VM from a source region is to create a RestorePointCollection in the target region.

#### URI Request
```
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}?api-version={api-version}
```
**NOTE:** api-version must be 2021-07-01 or later

#### Request Body
```
{ 
    "name": "name of the restorePointCollection resource", 
    "location": "location of the restorePointCollection resource",     
    "tags": { 
        "department": "finance" 
    }, 
    "properties": { 
         "source": { 
               “id”: "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/virtualMachines/{vmName}" 
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
    "name": "name of the restorePointCollection resource",
    "id": "CSM Id of restorePointCollection resource",
    "type": "Microsoft.Compute/restorePointCollections",
    "location": "location of the restorePointCollection resource",
    "tags": {
        "department": "finance"
    },
    "properties   ": {
        "restorePointCollectionId": "Guid Id of RestorePointCollection",
        "source": {
            "id": "/subscriptions/{subid}/resourceGroups/{resourceGroupName}/providers/microsoft.compute/virtualMachines/{vmName}",
            "location": "location of the virtualMachine resource"
        }
    }
} 
```

### Create VM Restore Point in Target Region
Next step is to trigger creation of a RestorePoint in the target RestorePointCollection.

#### URI request
```
PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?api-version={api-version}
```
#### Request body
```
{
    "name": "name of the restore point resource",
    "properties": {
        "excludeDisks": ["List of disks to be excluded in the RestorePoint"]
    }
}
```

**NOTE:** Location of the source VM would be inferred from that of the target RestorePointCollection

#### Response
The request response will include a status code and set of response headers. 

##### Status Code
This is a long running operation; hence the operation returns a 201 during create. The client is expected to poll for the status using the operation. (Both the Location and Azure-AsyncOperation headers are provided for this purpose.) 

During restore point creation, the ProvisioningState would appear as Creating in GET restore point API response. If creation fails, its ProvisioningState will be Failed. ProvisioningState would be set to Succeeded when the data copy across regions is initiated. 

**NOTE:** You can track the copy status by calling GET instance View (?$expand=instanceView) on the target VM Restore Point. Please check the "Get VM Restore Points Copy/Replication Status" section below on how to do this. VM Restore Point is considered usable (can be used to restore a VM) only when copy of all the disk restore points are successful.

##### Response Body
###### Intial Response Body
```
{
    "id": "CSM Id of the restore point",
    "name": "name of the restore point",
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
"optionalProperties": "opaque bag of properties to be passed to extension",
    "consistencyMode": "CrashConsistent | FileSystemConsistent | ApplicationConsistent",
    "provisioningState": "Succeeded | Failed | Creating | Deleting",
    "provisioningDetails": {
        "creationTime": "Creation Time of Restore point in UTC",
        "totalUsedSizeInBytes": "25",
        "statusCode": "status code reported by the extension",
        "statusMessage": "status message reported by the extension"
    }
}
```

###### Final Response Body
```
{
    "id": "CSM Id of the restore point",
    "name": "name of the restore point",
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
    "optionalProperties": "opaque bag of properties to be passed to extension",
    "consistencyMode": "CrashConsistent | FileSystemConsistent | ApplicationConsistent",
    "provisioningState": "Succeeded | Failed | Creating | Deleting",
    "provisioningDetails": {
        "creationTime": "Creation Time of Restore point in UTC",
        "totalUsedSizeInBytes": "25",
        "statusCode": "status code reported by the extension",
        "statusMessage": "status message reported by the extension"
    }
}
```


### Get VM Restore Points With Copy/Replication Status
Once creation of VM Restore Points is initiated, you can track the data copy status by calling GET instance View (?$expand=instanceView) on the VM Restore Point.

#### URI Request
```
GET https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Compute/restorePointCollections/{restorePointCollectionName}/restorePoints/{restorePointName}?$expand=instanceView&api-version={api-version}
```

#### Response
```
{
    "id": "CSM Id of the restore point",
    "name": "name of the restore point",
    "properties": {
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
"optionalProperties": "opaque bag of properties to be passed to extension",
        "consistencyMode": "CrashConsistent | FileSystemConsistent | ApplicationConsistent",
        "provisioningState": "Succeeded | Failed | Creating | Deleting",
        "provisioningDetails": {
            "creationTime": "Creation Time of Restore point in UTC",
            "totalUsedSizeInBytes": "25",
            "statusCode": "status code reported by the extension",
            "statusMessage": "status message reported by the extension"
        },
        "instanceView": {
            "statuses": [
                {
                    "code": "ReplicationState/succeeded",
                    "level": "Info",
                    "displayStatus": "Replication succeeded"
                }
            ],
            "diskRestorePoints": [
                {
                    "id": "<diskRestorePoint Arm Id>",
                    "replicationStatus": {
                        "status": {
                            "code": "ReplicationState/succeeded",
                            "level": "Info",
                            "displayStatus": "Replication succeeded"
                        },
                        "completionPercent": "<completion percentage of the replication>"
                    }
                }
            ]
        }
    }
}
```

## Known issues and limitations
* Cross-region creation functionality is only supported via REST APIs and template deployment. CLI, SDK, PS will be added later. 
* Concurrent creation of Restore Points for a VM in the same or different regions is not supported
* Error messages may not reflect the actual error that has occurred. We are working on improving the error messages to make them more intuitive.
* Restore point of CMK encrypted VM and disks will be encrypted using PMK in the target region. Customer needs to use CMK when restoring the disks and VM from the restore point.
* Currently, the replication progress in only updated once every 10mins. Hence for disks that have low churn, there can be scenarios where only the initial (0) and the terminal replication progress (100) can be seen.
* If there is huge amount data to be copied to the target region, depending on the bandwidth available between the regions, copy time could be couple of days. If time taken to copy the data exceeds 2 weeks, the restore points replication will be terminated and the Restore Point will not be usable.
* Restore points that are copied to the target region do not have a reference to the source VM. The source VM reference is available in the Restore Point Collection
* Billing of Restore Point Collection may not be accurate. You will see 2 seperate billing entries for a single Restore Point Collection (one entry in the source region and another in the target region).
* Other known issues and limitations of cross region copy of restore points may apply here as well
