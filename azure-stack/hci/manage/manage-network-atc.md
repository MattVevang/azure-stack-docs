---
title: Manage host networking with Network ATC
description: This topic covers how to manage host networking for Azure Stack HCI.
author: v-dasis
ms.topic: how-to
ms.date: 10/29/2021
ms.author: v-dasis
ms.reviewer: JasonGerend
---

# Manage host networking using Network ATC

> Applies to: Azure Stack HCI, version 21H2

This article discusses how to manage Network ATC after it has been deployed. Network ATC simplifies the deployment and network configuration management for Azure Stack HCI clusters. You use Windows PowerShell to manage Network ATC.

### Configure an override

You can modify the default configuration and verify that Network ATC makes the necessary changes.

> [!IMPORTANT]
> Network ATC implements the Microsoft-tested, **Best Practice** configuration. We highly recommend that you only modify the default configuration with guidance from Microsoft Azure Stack HCI support teams.

#### Update an intent with a single override

This task will help you override the default configuration which has already been deployed. This example modifies the default bandwidth reservation for SMB Direct.

> [!IMPORTANT]
> The `Set-NetIntent` cmdlet is used to update an already deployed intent. Use the `Add-NetIntent` cmdlet to add an override at initial deployment time

1. Get a list of possible override cmdlets. We use wildcards to see the options available:

    ```powershell
    Get-Command -Noun NetIntent*Over* -Module NetworkATC
    ```

1. Create an override object for the DCB Quality of Service (QoS) configuration:

    ```powershell
    $QosOverride = New-NetIntentQosPolicyOverrides
    $QosOverride
    ```

1. Modify the bandwidth percentage for SMB Direct:

    ```powershell
    $QosOverride.BandwidthPercentage_SMB = 25
    $QosOverride
    ```

    > [!NOTE]
    >It is expected that no values appear for any property you don’t override.

1. Submit the intent request specifying the override:

    ```powershell
    Set-NetIntent -Name Cluster_ComputeStorage -QosPolicyOverrides $QosOverride
    ```

1. Wait for the provisioning status to complete:

    ```powershell
    Get-NetIntentStatus -Name Cluster_ComputeStorage | Format-Table IntentName, Host, ProvisioningStatus, ConfigurationStatus
    ```

1. Check that the override has been properly set on all cluster nodes. In the example, the SMB_Direct traffic class was overridden with a bandwidth percentage of 25%:

    ```powershell
    Get-NetQosTrafficClass -Cimsession (Get-ClusterNode).Name | Select PSComputerName, Name, Priority, Bandwidth
    ```

#### Update an intent with multiple overrides

This task will help you override the default configuration which has already been deployed. This example modifies the default bandwidth reservation for SMB Direct and the maximum transmission unit (MTU) of the adapters.

1. Create an override object. In this example, we create two objects - one for QoS properties and one for a physical adapter property.

    ```powershell
    $QosOverride = New-IntentQosPolicyOverrides
    $AdapterOverride = New-NetIntentAdapterPropertyOverrides
    $QosOverride
    $AdapterOverride
    ```

1. Modify the SMB bandwidth percentage:

    ```powershell
    $QosOverride.BandwidthPercentage_SMB = 60
    $QosOverride
    ```

1. Modify the MTU size (JumboPacket) value:

    ```powershell
    $AdapterOverride.JumboPacket = 9014
    ```

1. Use the `Set-NetIntent` command to update the intent and specify the overrides objects previously created.

    Use the appropriate parameter based on the type of override you're specifying. In the example below, we use the `AdapterPropertyOverrides` parameter for the `$AdapterOverride` object that was created with `New-NetIntentAdapterPropertyOverrides` cmdlet whereas the `QosPolicyOverrides` parameter is used with the `$QosOverride` object created from `New-NetIntenQosPolicyOverrides` cmdlet.

    ```powershell
    Set-NetIntent -ClusterName HCI01 -Name Cluster_ComputeStorage -AdapterPropertyOverrides $AdapterOverride -QosPolicyOverride $QosOverride
    ```

1. First, notice that the status for all nodes in the cluster has changed to ProvisioningUpdate and Progress is on 1 of 2. The progress property is similar to a configuration watermark in that you have a new submission that must be enacted.

    ```powershell
    Get-NetIntentStatus -ClusterName HCI01
    ```

1. Wait for the provisioning status to complete:

    ```powershell
    Get-NetIntentStatus -ClusterName HCI01
    ```

1. Check that traffic class was overridden with a bandwidth percentage of 60%.

    ```powershell
    Get-NetQosTrafficClass -Cimsession (Get-ClusterNode).Name | Select PSComputerName, Name, Priority, Bandwidth 
    ```

1. Check that the adapters MTU (JumboPacket) value was modified and that the host virtual NICs created for storage also have been modified.

    ```powershell
    Get-NetAdapterAdvancedProperty -Name pNIC01, pNIC02, vSMB* -RegistryKeyword *JumboPacket -Cimsession (Get-ClusterNode).Name
    ```

    > [!IMPORTANT]
    > Ensure you modify the cmdlet above to include the adapter names in the intent specified.

## Remove an intent

If you want to test various configurations on the same adapters, you may need to remove an intent. If you previously deployed and configured Network ATC on your system, you may need to reset the node so that the configuration can be deployed. To do this, copy and paste the following commands to remove all existing intents and their corresponding vSwitch:

```powershell
    $intents = Get-NetIntent
    foreach ($intent in $intents)
    {
        Remove-NetIntent -Name $intent.IntentName
        Remove-VMSwitch -Name "*$($intent.IntentName)*" -ErrorAction SilentlyContinue -Force
    }
    
    Get-NetQosTrafficClass | Remove-NetQosTrafficClass
    Get-NetQosPolicy | Remove-NetQosPolicy -Confirm:$false
    Get-NetQosFlowControl | Disable-NetQosFlowControl
```

## Add a server node

You can add nodes to a cluster. Each node in the cluster receives the same intent, improving the reliability of the cluster. The new server node must meet all requirements as listed in the Requirements and best practices section of [Host networking with Network ATC](../deploy/network-atc.md).

In this task, you will add additional nodes to the cluster and observe how a consistent networking configuration is enforced across all nodes in the cluster.

1. Use the `Add-ClusterNode` cmdlet to add the additional (unconfigured) nodes to the cluster. You only need management access to the cluster at this time. Each node in the cluster should have all pNICs named the same.

    ```powershell
    Add-ClusterNode -Cluster HCI01
    Get-ClusterNode
    ```

1. Check the status across all cluster nodes using the `-ClusterName` parameter.

    ```powershell
    Get-NetIntentStatus -ClusterName HCI01
    ```

    > [!NOTE]
    > If pNICs do not exist on one of the additional nodes, `Get-NetIntentStatus` will report the error 'PhysicalAdapterNotFound', which easily identifies the provisioning issue.

1. Check the provisioning status of all nodes using `Get-NetIntentStatus`. The cmdlet reports the configuration for both nodes. Note that this may take a similar amount of time to provision as the original node.

    ```powershell
    Get-NetIntentStatus -ClusterName HCI01
    ```

You can also add several nodes to the cluster at once.

## Post-deployment tasks

There are several tasks to complete following a Network ATC deployment, including the following:

### Validate automatic remediation

Network ATC ensures that the deployed configuration stays the same across all cluster nodes. In this task, we modify one the configuration (without an override) emulating an accidental configuration change and observe how the reliability of the system is improved by remediating the misconfigured property.

>[!NOTE]
> ATC will automatically remediate all of the configuration it manages.

1. Check the adapter's existing MTU (JumboPacket) value:

    ```powershell
    Get-NetAdapterAdvancedProperty -Name pNIC01, pNIC02, vSMB* -RegistryKeyword *JumboPacket -Cimsession (Get-ClusterNode).Name
    ```

1. Modify one of the physical adapter's MTU without specifying an override. This emulates an accidental change or "configuration drift" which must be remediated.

    ```powershell
    Set-NetAdapterAdvancedProperty -Name pNIC01 -RegistryKeyword *JumboPacket -RegistryKeyword *JumboPacket -RegistryValue 4088
    ```

1. Verify that the adapter's existing MTU (JumboPacket) value has been modified:

    ```powershell
    Get-NetAdapterAdvancedProperty -Name pNIC01, pNIC02, vSMB* -RegistryKeyword *JumboPacket -Cimsession (Get-ClusterNode).Name
    ```

1. Retry the configuration. This step is only performed to expedite the remediation. Network ATC will automatically remediate this configuration.

    ```powershell
    Set-NetIntentRetryState -ClusterName HCI01 -Name Cluster_ComputeStorage
    ```

1. Verify that the consistency check has completed:

    ```powershell
    Get-NetIntentStatus -ClusterName HCI01 -Name Cluster_ComputeStorage
    ```

1. Verify that the adapter's MTU (JumboPacket) value has returned to the expected value:

    ```powershell
    Get-NetAdapterAdvancedProperty -Name pNIC01, pNIC02, vSMB* -RegistryKeyword *JumboPacket -Cimsession (Get-ClusterNode).Name
    ```

### Add non-APIPA addresses to storage adapters

This can be accomplished using DHCP on the storage VLANs or by using the `NetIPAddress` cmdlets.

### Set SMB bandwidth limits

If live migration uses SMB Direct (RDMA), configure a bandwidth limit to ensure that live migration does not consume all the bandwidth used by Storage Spaces Direct and Failover Clustering.

### Stretched cluster configuration

Stretched clusters require additional configuration that must be manually performed following the successful deployment of an intent. For stretched clusters, all nodes in the cluster must use the same intent.

## Next steps

- Learn more about [Stretched clusters](../concepts/stretched-clusters.md).
