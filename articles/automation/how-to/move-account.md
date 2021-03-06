---
title: Move your Azure Automation account to another subscription
description: This article describes how to move your Automation account to another subscription.
services: automation
ms.service: automation
ms.subservice: process-automation
author: mgoedtel
ms.author: magoedte
ms.date: 03/11/2019
ms.topic: conceptual
manager: carmonm
---
# Move your Azure Automation account to another subscription

Azure Automation allows you to move some resources to a new resource group or subscription. You can move resources through the Azure portal, PowerShell, the Azure CLI, or the REST API. To learn more about the process, see [Move resources to a new resource group or subscription](../../azure-resource-manager/management/move-resource-group-and-subscription.md).

The Azure Automation account is one of the resources that you can move. In this article, you'll learn to move Automation accounts to another resource or subscription. The high-level steps for moving your Automation account are:

1. Remove your solutions.
2. Unlink your workspace.
3. Move the Automation account.
4. Delete and recreate the Run As accounts.
5. Re-enable your solutions.

>[!NOTE]
>This article has been updated to use the new Azure PowerShell Az module. You can still use the AzureRM module, which will continue to receive bug fixes until at least December 2020. To learn more about the new Az module and AzureRM compatibility, see [Introducing the new Azure PowerShell Az module](https://docs.microsoft.com/powershell/azure/new-azureps-module-az?view=azps-3.5.0). For Az module installation instructions on your Hybrid Runbook Worker, see [Install the Azure PowerShell Module](https://docs.microsoft.com/powershell/azure/install-az-ps?view=azps-3.5.0). For your Automation account, you can update your modules to the latest version using [How to update Azure PowerShell modules in Azure Automation](../automation-update-azure-modules.md).

## Remove solutions

To unlink your workspace from your Automation account, you must remove these solutions from your workspace:

- Change Tracking and Inventory
- Update Management
- Start/Stop VMs during off hours

1. In the Azure portal, locate your resource group.
2. Find each solution and click **Delete** on the Delete Resources page.

    ![Delete solutions from the Azure portal](../media/move-account/delete-solutions.png)

    If you prefer, you can delete the solutions using the [Remove-AzResource](https://docs.microsoft.com/powershell/module/Az.Resources/Remove-AzResource?view=azps-3.7.0) cmdlet:

    ```azurepowershell-interactive
    $workspaceName = <myWorkspaceName>
    $resourceGroupName = <myResourceGroup>
    Remove-AzResource -ResourceType 'Microsoft.OperationsManagement/solutions' -ResourceName "ChangeTracking($workspaceName)" -ResourceGroupName $resourceGroupName
    Remove-AzResource -ResourceType 'Microsoft.OperationsManagement/solutions' -ResourceName "Updates($workspaceName)" -ResourceGroupName $resourceGroupName
    Remove-AzResource -ResourceType 'Microsoft.OperationsManagement/solutions' -ResourceName "Start-Stop-VM($workspaceName)" -ResourceGroupName $resourceGroupName
    ```

### Remove alert rules for the Start/Stop VMs during off hours solution

For the Start/Stop VMs during off hours solution, you also need to remove the alert rules created by the solution.

1. In the Azure portal, go to your resource group and select **Monitoring** > **Alerts** > **Manage alert rules**.

![Alerts page showing selection of Manage Alert rules](../media/move-account/alert-rules.png)

2. On the Rules page, you should see a list of the alerts configured in that resource group. The solution creates these rules:

    * AutoStop_VM_Child
    * ScheduledStartStop_Parent
    * SequencedStartStop_Parent

3. Select the rules one at a time and click **Delete** to remove them.

    ![Rules page requesting confirmation of deletion for selected rules](../media/move-account/delete-rules.png)

    > [!NOTE]
    > If you don't see any alert rules on the Rules page, change the **Status** field to Disabled to show disabled alerts, because you might have disabled them.

4. When the alert rules are removed, you must remove the action group created for Start/Stop VMs during off hours solution notifications. In the Azure portal, select **Monitor** > **Alerts** > **Manage action groups**.

5. Select **StartStop_VM_Notification**. 

6. On the action group page, select **Delete**.

    ![Action group page](../media/move-account/delete-action-group.png)

    If you prefer, you can delete your action group using the [Remove-AzActionGroup](https://docs.microsoft.com/powershell/module/az.monitor/remove-azactiongroup?view=azps-3.7.0) cmdlet:

    ```azurepowershell-interactive
    Remove-AzActionGroup -ResourceGroupName <myResourceGroup> -Name StartStop_VM_Notification
    ```

## Unlink your workspace

Now you can unlink your workspace:

1. In the Azure portal, select **Automation account** > **Related Resources** > **Linked workspace**. 

2. Select **Unlink workspace** to unlink the workspace from your Automation account.

    ![Unlink a workspace from an Automation account](../media/move-account/unlink-workspace.png)

## Move your Automation account

You can now move your Automation account and its runbooks. 

1. In the Azure portal, browse to the resource group of your Automation account. Select **Move** > **Move to another subscription**.

    ![Resource group page, move to another subscription](../media/move-account/move-resources.png)

2. Select the resources in your resource group that you want to move. Ensure that you include your Automation account, runbooks, and Log Analytics workspace resources.

## Recreate Run As accounts

[Run As accounts](../manage-runas-account.md) create a service principal in Azure Active Directory to authenticate with Azure resources. When you change subscriptions, the Automation account no longer uses the existing Run As account. To recreate the Run As accounts:

1. Go to your Automation account in the new subscription and select **Run as accounts** under **Account Settings**. You'll see that the Run As accounts show as incomplete now.

    ![Run As accounts are incomplete](../media/move-account/run-as-accounts.png)

2. Delete the Run As accounts one at a time using the **Delete** button on the Properties page. 

    > [!NOTE]
    > If you don't have permissions to create or view the Run As accounts, you see the following message: `You do not have permissions to create an Azure Run As account (service principal) and grant the Contributor role to the service principal.` To learn about the permissions required to configure a Run As account, see [Permissions required to configure Run As accounts](../manage-runas-account.md#permissions).

3. After you've deleted the Run As accounts, select **Create** under **Azure Run As account**. 

4. On the Add Azure Run As account page, select **Create** to create the Run As account and service principal. 

5. Repeat the steps above with the Azure Classic Run As account.

## Enable solutions

After you recreate the Run As accounts, you must re-enable the solutions that you removed before the move: 

1. To turn on the Change Tracking and Inventory solution, select Change Tracking and Inventory in your Automation account. Choose the Log Analytics workspace that you moved over and select **Enable**.

2. Repeat step 1 for the Update Management solution.

    ![Re-enable solutions in your moved Automation account](../media/move-account/reenable-solutions.png)

3. Machines that are onboarded with your solutions are visible when you've connected the existing Log Analytics workspace. To turn on the Start/Stop VMs during off hours solution, you must redeploy the solution. Under **Related Resources**, select **Start/Stop VMs** > **Learn more about and enable the solution** > **Create** to start the deployment.

4. On the Add Solution page, choose your Log Analytics workspace and Automation account.

    ![Add Solution menu](../media/move-account/add-solution-vm.png)

5. Configure the solution as described in [Start/Stop VMs during off hours solution in Azure Automation](../automation-solution-vm-management.md).

## Verify the move

When the move is complete, verify that the capabilities listed below are enabled. 

|Capability|Tests|Troubleshooting|
|---|---|---|
|Runbooks|A runbook can successfully run and connect to Azure resources.|[Troubleshoot runbooks](../troubleshoot/runbooks.md)
|Source control|You can run a manual sync on your source control repository.|[Source control integration](../source-control-integration.md)|
|Change tracking and inventory|Verify that you see current inventory data from your machines.|[Troubleshoot change tracking](../troubleshoot/change-tracking.md)|
|Update management|Verify that you see your machines and that they're healthy.</br>Run a test software update deployment.|[Troubleshoot update management](../troubleshoot/update-management.md)|
|Shared resources|Verify that you see all your shared resources, such as [credentials](../shared-resources/credentials.md), [variables](../shared-resources/variables.md), and the like.|

## Next steps

To learn more about moving resources in Azure, see [Move resources in Azure](../../azure-resource-manager/management/move-support-resources.md).
