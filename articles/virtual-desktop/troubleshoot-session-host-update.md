---
title: Troubleshoot session host update - Azure Virtual Desktop
description: Troubleshooting guidance to help with session host update.
ms.topic: troubleshooting
author: dknappettmsft
ms.author: daknappe
ms.date: 10/01/2024
---

# Troubleshoot session host update in Azure Virtual Desktop

> [!IMPORTANT]
> Session host update for Azure Virtual Desktop is currently in PREVIEW. This preview is provided as-is, with all faults and as available, and are excluded from the service-level agreements (SLAs) or any limited warranties Microsoft provides for Azure services in general availability.
>
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

Session host update in Azure Virtual Desktop enables you to easily update session host virtual machines (VMs) in a host pool with a session host configuration. This article helps troubleshoot some issues you could run into.

## Session host configuration failed to create when creating a host pool

When a session host configuration is created, the parameters provided for the configuration are checked during extended validation. Validation can fail if the service concludes that it will be unable to successfully create session hosts with the provided parameters. As the Azure resources are stored in your subscription, they can be modified by other processes; session host creation can still fail using the session host configuration even after this validation check is completed.

Here are some example failures:

- **VM availability**: the combination of VM SKU name, region, availability zone, and subscription isn't available. Some of the errors that can result include `VmSkuNotAvailableInRegion`, `VmSkuNotAvailableInRegionDueToRestriction`, and `AvailabilityZoneNotAvailable`. You need to review the availability of VM sizes and availability zones for your chosen region and subscription quota and provide a supported combination. Use the PowerShell cmdlet [`Get-AzComputeResourceSku`](/powershell/module/az.compute/get-azcomputeresourcesku) to identify the restrictions for a given combination of a VM SKU and region.

- **Parameter compatibility**: the combination of VM SKU, disk, image, and virtual network isn't compatible. Some of the errors that can result include `ComputeSkuIncompatibleWithImageHyperVGeneration`, `ImageDiskTypeIncompatible`, `VnetLocationIncompatible`. Review the [prerequisites for Azure Virtual Desktop](prerequisites.md) to ensure the provided parameters meet the requirements for session host creation.

If the session host configuration fails to create when creating a host pool, you aren't able to create a session host configuration for this host pool using the Azure portal. You can use PowerShell to create the session host configuration using the `New-AzWvdSessionHostConfiguration` cmdlet. Alternatively, you can delete the host pool and recreate it.

## Error: SessionHostConfiguration doesn't exist

If you get the error **Error: SessionHostConfiguration does not exist** when using the PowerShell cmdlet `Get-AzWvdSessionHostConfiguration`, create the session host configuration using the `New-AzWvdSessionHostConfiguration` cmdlet.

## Failed updates

When you update session hosts using session host update, it's possible that an individual session host fails to update. In this case, session host update attempts to roll back the update on that session host. The intention for the rollback is to maintain the capacity of the entire host pool, even though this session host is rolled back to a previous version of the session host configuration, rather than forcing the session host to be unavailable and reducing the capacity of the host pool. Other session hosts in the host pool that successfully updated aren't rolled back. Session hosts that didn't start updating aren't updated.

Once a session host fails to update, session host update completes updating the current batch of session hosts, then marks the update as failed. In this scenario, the only options are to retry the update or cancel it. If you retry the update, session host update again attempts to update the session that failed, plus the remaining session hosts not previously attempted. The existing batch size is used. If a session host fails to update a second time, it goes into an error state marked **Rollback-failed skipped** and is ignored for updates.

If a session host fails to roll back successfully, it isn't available to host session and capacity is reduced. The session host isn't the same as the other session hosts in the host pool and it match the session host configuration. You should investigate why the update of the session host failed and resolve the issue before scheduling a new update. Once you schedule a new update, session host update attempts to update the session host that failed so they all match, plus any session hosts that weren't started in the previous update attempt.

An update can fail with the following status:

| Status | Description |
|--|--|
| Update failed to initiate | The update flow is incorrect. For example, an image that's incompatible with the virtual machine SKU. You can't retry the update; you need to cancel it and schedule a new update. |
| Update failed | The update failed while it was in progress. If you retry the update, it continues with the session host it stopped at previously. |
| Session host rollback failed | If a session host fails to update, session host update tries to roll back the update on that session host. If the rollback fails and you retry the update, the session host is ignored and the update continues with the rest. |

You can get any errors for an update by following the steps to [Monitor the progress of an update](session-host-update-configure.md?tabs=powershell#monitor-the-progress-of-an-update). When you use Azure PowerShell, the variable `$updateProgress` contains error details in the following properties:

- `$updateProgress.PropertiesUpdateStatus`
- `$updateProgress.UpdateProgressError`
- `$updateProgress.UpdateProgressError.FaultText`

Once you identify the issue, you can either [retry the update, or cancel it and schedule a new update](session-host-update-configure.md#pause-resume-cancel-or-retry-an-update).

### An update failed to initiate

When a session host update is initiated, the service validates whether it will be able to successfully complete the update. When a session host update fails prior to starting, the update ends and changes can be made to the session host configuration to retry the update. As the Azure resources are stored in your subscription, they can be modified by other processes; session host creation can still fail using the session host configuration even after this validation check is completed.

Here are some example failures that prevent an update from starting:

- **No session hosts to update**: the error `HostpoolHasNoSessionHosts` is returned when there are no session hosts to update as part of the session host update. If you didn't make changes to the session host configuration prior to initiating an update, this error is returned.

- **Capacity issues**: validation checks for sufficient capacity in your virtual network subnet and VM core quota. This check does not guarantee capacity during an update; creation of other resources outside of session host update can result in errors mid-update associated with capacity limits. Set your batch size to be within the remaining quota for your subscription.

- **Parameter consistency with current session hosts**: session host update doesn't support changing the region, subscription, resource group, or domain join type for a session host. If the session host configuration contains properties in these fields that differ from the session hosts in the host pool, the update fails to start. You should remove the session hosts that are inconsistent with the configuration.

### Failures during an update

Here are some example failures that can happen during an update:

- **VM creation failures**: VM creation can fail for a variety of reasons not specific to Azure Virtual Desktop, for example the exhaustion of subscription capacity, or issues with the provided image. You should review the error message provided to determine the appropriate remediation. Open a support case with Azure support if you need further assistance.

- **Agent installation, domain join, and session host health errors or timeout**: in most cases, agent, domain join, and other session host health errors can be resolved by reviewing guidance for addressing deployment and domain join failures for Azure Virtual Desktop. In addition, you should ensure that your image doesn't have the PowerShell DSC extension installed in the image. If it is installed, remove the folder `C:\packages\plugin folder` from the image.

- **Resource modification and access errors**: modifying resources that are impacted in the update can result in errors during an update. Some of the errors that can result include deletion of resources and resource groups, changes to permissions, changes to power state, and changes to drain mode. In addition, if your Azure resources are locked and/or Azure policy limits the Azure Virtual Desktop service from modifying your session hosts, the update fails. Review Azure activity logs if you encounter related errors. Open a support case with Azure support if you need further assistance.

## Incompatible parameters passed to New-AzWvdSessionHostConfiguration

You can pass incompatible parameters to the `New-AzWvdSessionHostConfiguration` PowerShell cmdlet. For example, if you specify the parameter `DomainInfoJoinType` as **AzureActiveDirectory**, but also specify the parameter `ActiveDirectoryInfoDomainName` with an Active Directory domain name, the domain name is ignored without returning an error.

## Next steps

- [Example diagnostic queries for session host update in Azure Virtual Desktop](session-host-update-diagnostics.md)
