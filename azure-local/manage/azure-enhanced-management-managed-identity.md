---
title: Enhanced management of Azure Local from Azure
description: Learn how to use enhanced Azure management for Azure Local. This enhanced management is enabled via Managed Identity created for your Azure Local.
ms.topic: article
author: alkohli
ms.author: alkohli
ms.date: 06/10/2025
---

# Enhanced management of Azure Local from Azure

[!INCLUDE [hci-applies-to-22h2-later](../includes/hci-applies-to-22h2-later.md)]

This guide describes the feature in the May 2023 cumulative update for Azure Local, version 22H2, that enables enhanced management from Azure.


## About enhanced Azure management

In the May 2023 cumulative update for Azure Local, version 22H2, a feature enhancement has been made to the operating system that enables additional capabilities for Azure Local to be managed from Azure.

This feature enhancement includes support of managed identity for Azure Local resources in Azure to enable Azure services such as Azure Monitor and Azure Site Recovery. The managed identity is created when your Azure Local is registered with Azure and persists for the lifetime of resource in Azure. The managed identity communicates with the resource provider in Azure and is used to authenticate your Azure Local with Azure.

This feature also includes Azure Service Bus integration to allow for an improved user experience managing your Azure Local from Azure.

## Benefits

The managed identity serves as an identity for the various components of your machine to authenticate with Azure and enables support for the following scenarios:

- **Monitor Azure Local Insights with Azure Monitor Agent** – The enhanced Azure Local Insights feature in Azure Monitor requires the Azure Monitor Agent instead of the legacy Microsoft Monitoring Agent (MMA). The Azure Monitor Agent uses managed identity to send logs and data to your Log Analytics workspace.

    For more information, see [Monitor Azure Local with Azure Monitor Insights](./monitor-single-23h2.md).

- **Protection of VM workloads via Azure Site Recovery** - You can protect your business critical virtual machine (VM) workloads running on Azure Local by replicating the VMs using the Azure Site Recovery agent which is deployed as an Arc for Server extension. The Azure Local managed identity is used to download a key credential file from Azure. This file lets the agent know which service to talk to and which Recovery services vault to communicate with. This mechanism allows us to scope the access to the Recovery services vault to only the applicable Azure Local.

    The Arc for Server extension uses the system managed identity to download the key credential file to every machine of the system. If a new machine is added to your system, Azure Local automatically triggers the installation of Arc for Server extension for Azure Site Recovery on the new machine. In the absence of managed identity, this was previously a manual step that required you to install the agent to each machine that was added to the system.

    For more information, see [Protect VM workloads with Azure Site Recovery on Azure Local](./azure-site-recovery.md).

With this feature enhancement, the following actions can be initiated from Azure and applied to the Azure Local within seconds:

- Enabling a Windows Server guest license subscription via the Azure portal. For more information, see [Activate Windows Server VMs on Azure Local](../manage/vm-activate.md?tab=azure-portal#enable-windows-server-subscription).
- Changing the level of service health data sent to Microsoft. For more information, see [Azure Local data collection](../concepts/data-collection.md) to understand the diagnostic data that Microsoft collects for your system.

## Enable enhanced management

To enable the enhanced management feature, you'll need to install the latest cumulative update for Azure Local, version 22H2 and rerun registration for your system.

## Prerequisites

Before you begin, make sure to complete the following prerequisites.

To reach the Azure Service Bus endpoint required by this feature, include the following URL to your outbound firewall allowlist:

- URL: *servicebus.windows.net*
- Ports: *443, 5671, 5672*

## Enable enhanced management for Azure Local, version 22H2

For systems running version 22H2, to enable Azure management and managed identity, follow these steps:

1. Install the May 2023 cumulative update for Azure Local, version 22H2.

1. On one of the machines, install or update to the latest `Az.StackHCI` PowerShell module that includes the latest registration script changes.
    - To install the module, run the following command in PowerShell:

        ```powershell
        Install-Module -Name Az.StackHCI
        ```

    - To update the module, run the following command in PowerShell:

        ```powershell
        Update-Module -Name Az.StackHCI
        ```

1. Skip this step and go to the next step if your system is already registered. If your system hasn't been previously registered to Azure, [register your system with Azure](../deploy/register-with-azure.md). The registration process configures a managed identity and Azure Service Bus to enable the new management feature.
1. If the system is already registered to Azure, rerun the registration. Use of the `RepairRegistration` parameter helps configure a managed identity and Azure Service Bus while retaining other information such as resource name, resource group, and other settings.

    ```powershell
    Register-AzStackHCI  -SubscriptionId "<subscription_ID>" -RepairRegistration
    ```

> [!NOTE]
> The registration fails if you use an older version, earlier than 1.4.1 for your `Az.StackHCI` PowerShell module. The updated module is backward compatible and runs on OS versions with or without the new feature update installed.


## Next steps

[Learn more about how to protect Azure Local VM workloads with Azure Site Recovery](./azure-site-recovery.md)
