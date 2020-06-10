# Welcome to DevTest Lab

![enter image description here](content/Microsoft.png)

## Conditions and Terms of Use Microsoft Confidential - For Internal Use Only

This training package is proprietary and confidential, and is intended only for uses described in the training materials. Content and software is provided to you under a Non-Disclosure Agreement and cannot be distributed. Copying or disclosing all or any portion of the content and/or software included in such packages is strictly prohibited.

The contents of this package are for informational and training purposes only and are provided "as is" without warranty of any kind, whether express or implied, including but not limited to the implied warranties of merchantability, fitness for a particular purpose, and non-infringement.

Training package content, including URLs and other Internet Web site references, is subject to change without notice. Because Microsoft must respond to changing market conditions, the content should not be interpreted to be a commitment on the part of Microsoft, and Microsoft cannot guarantee the accuracy of any information presented after the date of publication. Unless otherwise noted, the companies, organizations, products, domain names, e-mail addresses, logos, people, places, and events depicted herein are fictitious, and no association with any real company, organization, product, domain name, e-mail address, logo, person, place, or event is intended or should be inferred.

**Copyright and Trademarks**

Microsoft may have patents, patent applications, trademarks, copyrights, or other intellectual property rights covering subject matter in this document. Except as expressly provided in written license agreement from Microsoft, the furnishing of this document does not give you any license to these patents, trademarks, copyrights, or other intellectual property.

Complying with all applicable copyright laws is the responsibility of the user. Without limiting the rights under copyright, no part of this document may be reproduced, stored in or introduced into a retrieval system, or transmitted in any form or by any means (electronic, mechanical, photocopying, recording, or otherwise), or for any purpose, without the express written permission of Microsoft Corporation.

For more information, see Use of Microsoft Copyrighted Content at [(http://www.microsoft.com/about/legal/permissions/)](http://www.microsoft.com/about/legal/permissions/)

Microsoft®, Internet Explorer®, and Windows® are either registered trademarks or trademarks of Microsoft Corporation in the United States and/or other countries. Other Microsoft products mentioned herein may be either registered trademarks or trademarks of Microsoft Corporation in the United States and/or other countries. All other trademarks are property of their respective owners.

© 2019 Microsoft Corporation.  All rights reserved.

# Start virtual machines in a lab in order by using Azure Automation runbooks

### In this article

1.  [Setup](#setup)
2.  [Create an Azure Automation account](#create-an-azure-automation-account)
3.  [Add a runbook](#add-a-runbook)
4.  [PowerShell script](#powershell-script)
5.  [Create a schedule](#create-a-schedule)


The [autostart](devtest-lab-set-lab-policy#set-autostart) feature of DevTest Labs allows you to configure VMs to start automatically at a specified time. However, this feature does not support machines to start in a specific order. There are several scenarios where this type of automation would be useful. One scenario is where a Jumpbox VM within a lab needs to be started first, before the other VMs, as the Jumpbox is used as the access point to the other VMs. This article shows you how to set up an Azure Automation account with a PowerShell runbook that executes a script. The script uses tags on VMs in the lab to allow you to control the startup order without having to change the script.

## Setup[](#setup)

In this example, VMs in the lab need to have the tag **StartupOrder** added with the appropriate value (0,1,2, etc.). Designate any machine that doesn't need to be started as -1.

## Create an Azure Automation account[](#create-an-azure-automation-account)

Create an Azure Automation account by following instructions in [this article](../automation/automation-create-standalone-account). Choose the **Run As Accounts** option when creating the account. Once the automation account is created, open the **Modules** page, and select **Update Azure Modules** on the menu bar. The default modules are several versions old and without the update the script may not function.

## Add a runbook[](#add-a-runbook)

Now, to add a runbook to the automation account, select **Runbooks** on the left menu. Select **Add a runbook** on the menu, and follow instructions to [create a PowerShell runbook](../automation/automation-first-runbook-textual-powershell).

## PowerShell script[](#powershell-script)

The following script takes the subscription name, the lab name as parameters. The flow of the script is to get all the VMs in the lab, and then parse out the tag information to create a list of the VM names and their startup order. The script walks through the VMs in order and starts the VMs. If there are multiple VMs in a specific order number, they are started asynchronously using PowerShell jobs. For those VMs that don’t have a tag, set startup value to be the last (10), they will be started last, by default. If the lab doesn’t want the VM to be autostarted, set the tag value to 11 and it will be ignored.


    #Requires -Version 3.0
    #Requires -Module AzureRM.Resources

    param
    (
        [Parameter(Mandatory=$false, HelpMessage="Name of the subscription that has the lab")]
        [string] $SubscriptionName,

        [Parameter(Mandatory=$false, HelpMessage="Lab name")]
        [string] $LabName
    )

    # Connect and add the appropriate subscription
    $Conn = Get-AutomationConnection -Name AzureRunAsConnection

    Add-AzureRMAccount -ServicePrincipal -Tenant $Conn.TenantID -ApplicationID $Conn.ApplicationId -Subscription $SubscriptionName -CertificateThumbprint $Conn.CertificateThumbprint

    # Find the lab
    $dtLab = Find-AzResource -ResourceType 'Microsoft.DevTestLab/labs' -ResourceNameEquals $LabName

    # Get the VMs
    $dtlAllVms = New-Object System.Collections.ArrayList
    $AllVMs = Get-AzResource -ResourceId "$($dtLab.ResourceId)/virtualmachines" -ApiVersion 2016-05-15

    # Get the StartupOrder tag, if missing set to be run last (10)
    ForEach ($vm in $AllVMs) {
        if ($vm.Tags) {
            if ($vm.Tags['StartupOrder']) {
                $startupValue = $vm.Tags['StartupOrder']
            } else {
                $startupValue = 10
            }
            } else {
                $startupValue = 10
            }
            $dtlAllVms.Add(@{$vm.Name = $startupValue}) > $null
    }

    # Setup for the async multiple vm start

    # Save profile
    $profilePath = Join-Path $env:Temp "profile.json"
    If (Test-Path $profilePath){
        Remove-Item $profilePath
    }
    Save-AzContext -Path $profilePath

    # Job to start VMs asynch
    $startVMBlock = {
        Param($devTestLab,$vmToStart,$profilePath)
        Import-AzContext -Path ($profilePath)
        Invoke-AzResourceAction `
            -ResourceId "$($devTestLab.ResourceId)/virtualmachines/$vmToStart" `
            -Action Start `
            -Force
        Write-Output "Started: $vmToStart"
    }

    $current = 0
    # Start in order from 0 to 10

    While ($current -le 10) {
    # Get the VMs in the current stage
        $tobeStarted = $null
        $tobeStarted = $dtlAllVms | Where-Object { $_.Values -eq $current}
        if ($tobeStarted.Count -eq 1) {
            # Run sync – jobs not necessary for a single VM
            $returnStatus = Invoke-AzResourceAction `
                    -ResourceId "$($dtLab.ResourceId)/virtualmachines/$($tobeStarted.Keys)" `
                    -Action Start `
                    -Force
            Write-Output "$($tobeStarted.Keys) status: $($returnStatus.status)"
        } elseif ($tobeStarted.Count -gt 1) {
            # Start multiple VMs async
            $jobs = @()
            Write-Output "Start Jobs start: $(Get-Date)"

            # Jobs
            $jobs += Start-Job -ScriptBlock $startVMBlock -ArgumentList $dtLab, $($singlevm.Keys), $profilePath
            Write-Output "Start Jobs end: $(Get-Date)"
        }

        # Get results from all jobs
        if($jobs.Count -ne 0) {
            Write-Output "Receive Jobs start: $(Get-Date)"
            foreach ($job in $jobs){
                $jobResult = Receive-Job -Job $job -Wait | Write-Output
            }
            Remove-Job -Job $jobs -Force
        }
        else
        {
            Write-Output "Information: No jobs available"
        }
    }

## Create a schedule[](#create-a-schedule)

To have this script execute daily, [create a schedule](../automation/shared-resources/schedules#creating-a-schedule) in the automation account. Once the schedule is created, [link it to the runbook](../automation/shared-resources/schedules#linking-a-schedule-to-a-runbook).

In a large-scale situation where there are multiple subscriptions with multiple labs, store the parameter information in a file for different labs and pass the file to the script instead of the individual parameters. The script would need to be modified but the core execution would be the same. While this sample uses the Azure Automation to execute the PowerShell script, there are other options like using a task in a Build/Release pipeline.

