---
title: "Tutorial: Try Azure Virtual Desktop with Windows 11"
description: This tutorial shows you how to deploy Azure Virtual Desktop with a Windows 11 desktop in a sample infrastructure with the Azure portal.
ms.topic: tutorial
author: dknappettmsft
ms.author: daknappe
ms.date: 10/25/2023
---

# Tutorial: Deploy a sample Azure Virtual Desktop infrastructure with a Windows 11 desktop

Azure Virtual Desktop enables you to access desktops and applications from virtually anywhere. This tutorial shows you how to deploy a *Windows 11 Enterprise* desktop in Azure Virtual Desktop using the Azure portal and how to connect to it. To learn more about the terminology used for Azure Virtual Desktop, see [Azure Virtual Desktop terminology](environment-setup.md) and [What is Azure Virtual Desktop?](overview.md)

You'll deploy a sample infrastructure by:

> [!div class="checklist"]
> * Creating a personal host pool.
> * Creating a session host virtual machine (VM) joined to your Microsoft Entra tenant with Windows 11 Enterprise and add it to the host pool.
> * Creating a workspace and an application group that publishes a desktop to the session host VM.
> * Assigning users to the application group.
> * Connecting to the desktop.

> [!TIP]
> This tutorial shows a simple way you can get started with Azure Virtual Desktop. It doesn't provide an in-depth guide of the different options and you can't publish a RemoteApp in addition to the desktop. For a more in-depth and adaptable approach to deploying Azure Virtual Desktop, see [Deploy Azure Virtual Desktop](deploy-azure-virtual-desktop.md), or for suggestions of what else you can configure, see the articles we list in [Next steps](#next-steps).

## Prerequisites

You need:

- An Azure account with an active subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

- The Azure account must be assigned the following built-in role-based access control (RBAC) roles as a minimum on the subscription, or on a resource group. For more information, see [Assign Azure roles using the Azure portal](../role-based-access-control/role-assignments-portal.yml). If you want to assign the roles to a resource group, you need to create this first.

   | Resource type | RBAC role |
   |--|--|
   | Host pool, workspace, and application group | [Desktop Virtualization Contributor](rbac.md#desktop-virtualization-contributor) |
   | Session hosts | [Virtual Machine Contributor](../role-based-access-control/built-in-roles.md#virtual-machine-contributor) |

   Alternatively if you already have the [Contributor](../role-based-access-control/built-in-roles.md#contributor) or [Owner](../role-based-access-control/built-in-roles.md#owner) RBAC role, you're already able to create all of these resource types.

- A [virtual network](../virtual-network/quick-create-portal.md) in the same Azure region you want to deploy your session hosts to.

- A user account in Microsoft Entra ID you can use for connecting to the desktop. This account must be assigned the *Virtual Machine User Login* or *Virtual Machine Administrator Login* RBAC role on the subscription. Alternatively you can assign the role to the account on the session host VM or the resource group containing the VM after deployment.

- A Remote Desktop client installed on your device to connect to the desktop. You can find a list of supported clients in [Remote Desktop clients for Azure Virtual Desktop](users/remote-desktop-clients-overview.md). Alternatively you can use the [Remote Desktop Web client](users/connect-web.md), which you can use through a supported web browser without installing any extra software.

## Create a personal host pool, workspace, application group, and session host VM

To create a personal host pool, workspace, application group, and session host VM running Windows 11:

1. Sign in to the [Azure portal](https://portal.azure.com).

1. In the search bar, type *Azure Virtual Desktop* and select the matching service entry.

1. From the Azure Virtual Desktop overview page, select **Create a host pool**.

1. On the **Basics** tab, complete the following information:

   | Parameter | Value/Description |
   |--|--|
   | **Project details** |  |
   | Subscription | Select the subscription you want to deploy your host pool, session hosts, workspace, and application group in from the drop-down list. |
   | Resource group | Select an existing resource group or select **Create new** and enter a name. |
   | Host pool name | Enter a name for the host pool, for example **hp01**. |
   | Location | Select the Azure region from the list where you want to create your host pool, workspace, and application group. |
   | Validation environment | Select **No**. This setting enables your host pool to receive service updates before all other production host pools, but isn't needed for this tutorial.|
   | Preferred app group type | Select **Desktop**. With this personal host pool, you publish a desktop, but you can't also add a RemoteApp application group for the same host pool to also publish applications. See [Next steps](#next-steps) for more advanced scenarios. |
   | **Host pool type** |  |
   | Host pool type | Select **Personal**. This means that end users have a dedicated assigned session host that they always connect to. Selecting **Personal** shows a new option for **Assignment type**. |
   | Assignment type | Select **Automatic**. Automatic assignment means that a user automatically gets assigned the first available session host when they first sign in, which is then dedicated to that user. |

   Once you've completed this tab, select **Next: Virtual Machines**.

1. On the **Virtual machines** tab, complete the following information:

   | Parameter | Value/Description |
   |--|--|
   | Add Azure virtual machines | Select **Yes**. This shows several new options. |
   | Resource group | This automatically defaults to the resource group you chose your host pool to be in on the *Basics* tab. |
   | Name prefix | Enter a name for your session hosts, for example **hp01-sh**.<br /><br />This name prefix is used as the prefix for your session host VMs. Each session host has a suffix of a hyphen and then a sequential number added to the end, for example **hp01-sh-0**.<br /><br />The prefix can be a maximum of 11 characters and is used in the computer name in the operating system. The prefix and the suffix combined can be a maximum of 15 characters. Session host names must be unique. |
   | Virtual machine location | Select the Azure region where you want to deploy your session host VMs. It must be the same region that your virtual network is in. |
   | Availability options | Select **No infrastructure redundancy required**. This means that your session host VMs aren't deployed in an availability set or in availability zones. |
   | Security type | Select **Trusted launch virtual machines**. Leave the subsequent defaults of **Enable secure boot** and **Enable vTPM** checked, and **Integrity monitoring** unchecked. For more information, see [Trusted launch](security-guide.md#trusted-launch). |
   | Image | Select **Windows 11 Enterprise, version 22H2**. |
   | Virtual machine size | Accept the default SKU. If you want to use a different SKU, select **Change size**, then select from the list. |
   | Number of VMs | Enter **1** as a minimum. You can deploy up to 500 session host VMs at this point if you wish, or you can add more separately.<br /><br />With a personal host pool, each session host can only be assigned to one user, so you need one session host for each user connecting to this host pool. Once you've completed this tutorial, you can create a pooled host pool, where multiple users can connect to the same session host. |
   | OS disk type | Select **Premium SSD** for best performance. |
   | Boot Diagnostics | Select **Enable with managed storage account (recommended)**. |
   | **Network and security** |  |
   | Virtual network | Select your virtual network and subnet to connect session hosts to. |
   | Network security group | Select **Basic**. |
   | Public inbound ports | Select **No** as you don't need to open inbound ports to connect to Azure Virtual Desktop. Learn more at [Understanding Azure Virtual Desktop network connectivity](network-connectivity.md). |
   | **Domain to join** |  |
   | Select which directory you would like to join | Select **Microsoft Entra ID**. |
   | Enroll VM with Intune | Select **No.** |
   | **Virtual Machine Administrator account** |  |
   | Username | Enter a name to use as the local administrator account for these session host VMs. |
   | Password | Enter a password for the local administrator account. |
   | Confirm password | Reenter the password. |
   | **Custom configuration** |  |
   | Custom configuration script URL | Leave this blank. |

   Once you've completed this tab, select **Next: Workspace**.

1. On the **Workspace** tab, complete the following information:

   | Parameter | Value/Description |
   |--|--|
   | Register desktop app group | Select **Yes**. This registers the default desktop application group to the selected workspace. |
   | To this workspace | Select **Create new** and enter a name, for example **ws01**. |

   Once you've completed this tab, select **Next: Review + create**. You don't need to complete the other tabs.

1. On the **Review + create** tab, ensure validation passes and review the information that is used during deployment. If validation doesn't pass, review the error message and check what you entered in each tab.

1. Select **Create**. A host pool, workspace, application group, and session host are created. Once your deployment is complete, select **Go to resource** to go to the host pool overview.

1. Finally, from the host pool overview, select **Session hosts** and verify the status of the session hosts is **Available**.

## Assign users to the application group

Once your host pool, workspace, application group, and session host VM(s) have been deployed, you need to assign users to the application group that was automatically created. After users are assigned to the application group, they'll automatically be assigned to an available session host VM because *Assignment type* was set to **Automatic** when the host pool was created.

1. From the host pool overview, select **Application groups**.

1. Select the application group from the list, for example **hp01-DAG**.

1. From the application group overview, select **Assignments**.

1. Select **+ Add**, then search for and select the user account you want to be assigned to this application group.

1. Finish by selecting **Select**.

## Enable connections from Remote Desktop clients

> [!TIP]
> This section is optional if you're going to use a Windows device to connect to Azure Virtual Desktop that is joined to the same Microsoft Entra tenant as your session host VMs and you're using the [Remote Desktop client for Windows](users/connect-windows.md?toc=%2Fazure%2Fvirtual-desktop%2Ftoc.json).

To enable connections from all of the Remote Desktop clients, you need to add an RDP property to your host pool configuration.

1. Go back to the host pool overview, then select **RDP Properties**.

1. Select the **Advanced** tab.

1. In the **RDP Properties** box, add `targetisaadjoined:i:1;` to the start of the text in the box.

1. Select **Save**.

## Connect to the desktop

You're ready to connect to the desktop. The desktop takes longer to load the first time as the profile is being created, however subsequent connections are quicker.

> [!IMPORTANT]
> Make sure the user account you're using to connect has been assigned the *Virtual Machine User Login* or *Virtual Machine Administrator Login* RBAC role on the subscription, session host VM, or the resource group containing the VM, as mentioned in the prerequisites, else you won't be able to connect.

Select the relevant tab and follow the steps, depending on which Remote Desktop client you're using. We've only listed the steps here for Windows, Web and macOS, but if you want to connect using one of our other Remote Desktop clients, see [Remote Desktop clients for Azure Virtual Desktop](users/remote-desktop-clients-overview.md).

# [Windows](#tab/windows-client)

1. Open the **Remote Desktop** app on your device.

1. Select the three dots in the top right-hand corner, then select **Subscribe with URL**.

1. In the **Email or Workspace URL** box, enter `https://rdweb.wvd.microsoft.com`. After a few seconds, the message **We found Workspaces at the following URLs** should be displayed.

1. Select **Next**.

1. Sign in with the user account you assigned to the application group. After a few seconds, the workspace should show with an icon named **SessionDesktop**.

1. Double-click **SessionDesktop** to launch a desktop session. You need to enter the password for the user account again.

# [Web](#tab/web-client)

1. Open a [supported web browser](users/connect-web.md#prerequisites) and go to [**https://client.wvd.microsoft.com/arm/webclient**](https://client.wvd.microsoft.com/arm/webclient).

1. Sign in with the user account you assigned to the application group. After a few seconds, the workspace should show with an icon named **SessionDesktop**.

1. Select **SessionDesktop** to launch a desktop session.

1. A prompt shows asking you for permission to **Access local resources**. You can also select whether you want to allow access to your microphone, clipboard, printer, and/or file transfer in the remote session. Once you've made your selections, select **Allow**.  If you allowed access to the microphone and/or clipboard, an extra prompt shows requesting further confirmation. 

1. The **Log in** prompt shows with your username prepopulated. You need to enter the password for the user account again, then select **Submit**.

# [macOS](#tab/macos-client)

1. Open the **Microsoft Remote Desktop** app on your device.

1. Select **Workspaces**.

1. Select **+**, then select **Add Workspace**.

1. In the **Email or Workspace URL** box, enter `https://rdweb.wvd.microsoft.com`. After a few seconds, the message **We found Workspaces at the following URLs** should be displayed.

1. Select **Add**.

1. Sign in with the user account you assigned to the application group. After a few seconds, the workspace should show with an icon named **SessionDesktop**.

1. Double-click **SessionDesktop** to launch a desktop session. You need to enter the password for the user account again, then select **Continue**.

---

## Next steps

Now that you've created and connected to a Windows 11 desktop with Azure Virtual Desktop there's much more you can do. For a more in-depth and adaptable approach to deploying Azure Virtual Desktop, see [Deploy Azure Virtual Desktop](deploy-azure-virtual-desktop.md), or for suggestions of what else you can configure, see:

- [Add session hosts to a host pool](add-session-hosts-host-pool.md).

- [Publish applications](manage-app-groups.md).

- [User profile management for Azure Virtual Desktop with FSLogix profile containers](create-profile-container-azure-ad.yml).

- [Understand network connectivity](network-connectivity.md).

- Learn about [supported identities and authentication methods](authentication.md)

- [Set up email discovery to subscribe to Azure Virtual Desktop](/windows-server/remote/remote-desktop-services/rds-email-discovery?toc=%2Fazure%2Fvirtual-desktop%2Ftoc.json).

- [Configure single sign-on for Azure Virtual Desktop using Microsoft Entra authentication](configure-single-sign-on.md).

- Learn about [session host virtual machine sizing guidelines](/windows-server/remote/remote-desktop-services/virtual-machine-recs?toc=%2Fazure%2Fvirtual-desktop%2Ftoc.json).

- [Use Microsoft Teams on Azure Virtual Desktop](teams-on-avd.md).

- [Monitor your deployment with Azure Virtual Desktop Insights](azure-monitor.md).
