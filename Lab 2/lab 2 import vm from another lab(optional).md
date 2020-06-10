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

# Import virtual machines from another lab in Azure DevTest Labs

### In this article

1.  [Scenarios](#scenarios)
2.  [Solution and constraints](#solution-and-constraints)
3.  [Use PowerShell](#use-powershell)
4.  [Use HTTP REST to import a VM](#use-http-rest-to-import-a-vm)


This article provides information about how to import virtual machines from another lab into your lab.

## Scenarios[](#scenarios)

Here are some scenarios where you need to import VMs from one lab into another lab:

*   An individual on the team is moving to another group within the enterprise and wants to take the developer desktop to the new team’s DevTest Labs.
*   The group has hit a [subscription-level quota](../azure-subscription-service-limits) and wants to split up the teams into a few subscriptions
*   The company is moving to Express Route (or some other new networking topology) and the team wants to move the Virtual Machines to use this new infrastructure

## Solution and constraints[](#solution-and-constraints)

This feature enables you to import VMs in one lab (source) into another lab (destination). You can optionally give a new name for the destination VM in the process. The import process includes all the dependencies like disks, schedules, network settings, and so on.

The process does take some time and is impacted by the following factors:

*   Number/size of the disks that are attached to the source machine (since it’s a copy operation and not a move operation)
*   Distance to the destination (For example, East US region to Southeast Asia).

Once the process is complete, the source Virtual Machine remains shutdown and the new one is running in the destination lab.

There are two key constraints to be aware of when planning to import VMs from one lab in to another lab:

*   Virtual Machine imports across subscriptions and across regions are supported, but the subscriptions must be associated to the same Azure Active Directory tenant.
*   Virtual Machines must not be in a claimable state in the source lab.
*   You're the owner of the VM in the source lab and owner of the lab in the destination lab.
*   Currently, this feature is supported only through Powershell and REST API.

## Use PowerShell[](#use-powershell)

Download ImportVirtualMachines.ps1 file from the [GitHub](https://github.com/Azure/azure-devtestlab/tree/master/samples/DevTestLabs/Scripts/ImportVirtualMachines). You can use the script to import a single VM or all VMs in the source lab into the destination lab.

### Use PowerShell to import a single VM[](#use-powershell-to-import-a-single-vm)

Executing this powershell script requires identifying the source VM and the destination lab, and optionally supplying a new name to use for the destination machine:



    ./ImportVirtualMachines.ps1 -SourceSubscriptionId "<ID of the subscription that contains the source lab>" `
                                -SourceDevTestLabName "<Name of the source lab>" `
                                -SourceVirtualMachineName "<Name of the VM to be imported from the source lab> " `
                                -DestinationSubscriptionId "<ID of the subscription that contians the destination lab>" `
                                -DestinationDevTestLabName "<Name of the destination lab>" `
                                -DestinationVirtualMachineName "<Optional: specify a new name for the imported VM in the destination lab>"

### Use PowerShell to import all VMs in the source lab[](#use-powershell-to-import-all-vms-in-the-source-lab)

If the Source Virtual Machine isn’t specified, the script automatically imports all VMs in the DevTest Labs. For example:



    ./ImportVirtualMachines.ps1 -SourceSubscriptionId "<ID of the subscription that contains the source lab>" `
                                -SourceDevTestLabName "<Name of the source lab>" `
                                -DestinationSubscriptionId "<ID of the subscription that contians the destination lab>" `
                                -DestinationDevTestLabName "<Name of the destination lab>"

## Use HTTP REST to import a VM[](#use-http-rest-to-import-a-vm)

The REST call is simple. You give enough information to identify the source and destination resources. Remember that the operation takes place on the destination lab resource.


    POST https://management.azure.com/subscriptions/<DestinationSubscriptionID>/resourceGroups/<DestinationResourceGroup>/providers/Microsoft.DevTestLab/labs/<DestinationLab>/ImportVirtualMachine?api-version=2017-04-26-preview
    {
       sourceVirtualMachineResourceId: "/subscriptions/<SourceSubscriptionID>/resourcegroups/<SourceResourceGroup>/providers/microsoft.devtestlab/labs/<SourceLab>/virtualmachines/<NameofVMTobeImported>",
       destinationVirtualMachineName: "<NewNameForImportedVM>"
    }
