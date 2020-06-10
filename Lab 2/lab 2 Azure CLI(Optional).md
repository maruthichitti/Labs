# Welcome to DevTest Labs

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

# Create and manage virtual machines with DevTest Labs using the Azure CLI

### In this article

1.  [Create and verify the virtual machine](#create-and-verify-the-virtual-machine)
2.  [Verify that the VM is available.](#verify-that-the-vm-is-available)
3.  [Start and connect to the virtual machine](#start-and-connect-to-the-virtual-machine)
4.  [Update the virtual machine](#update-the-virtual-machine)
5.  [Stop and delete the virtual machine](#stop-and-delete-the-virtual-machine)
6.  [Next steps](#next-steps)

This quickstart will guide you through creating, starting, connecting, updating, and cleaning up a development machine in your lab.

Before you begin:

*   If a lab has not been created, instructions can be found [here](devtest-lab-create-lab).

*   [Install the Azure CLI](/en-us/cli/azure/install-azure-cli). To start, run az login to create a connection with Azure.

## Create and verify the virtual machine[](#create-and-verify-the-virtual-machine)

Before you execute DevTest Labs related commands, set the appropriate Azure context by using the `az account set` command:


    az account set --subscription 11111111-1111-1111-1111-111111111111

The command to create a virtual machine is: `az lab vm create`. The resource group for the lab, lab name, and virtual machine name are all required. The rest of the arguments change depending on the type of virtual machine.

The following command creates a Windows-based image from Azure Market Place. The name of the image is the same as you would see when creating a virtual machine using the Azure portal.

<div class="codeHeader" id="code-try-1" data-bi-name="code-header"><span class="language">Azure CLI</span> <button class="action" data-bi-name="copy" aria-label="Copy code"><span class="docon docon-edit-copy" role="presentation"></span><span>Copy</span></button></div>

    az lab vm create --resource-group DtlResourceGroup --lab-name MyLab --name 'MyTestVm' --image "Visual Studio Community 2017 on Windows Server 2016 (x64)" --image-type gallery --size 'Standard_D2s_v3’ --admin-username 'AdminUser' --admin-password 'Password1!'

The following command creates a virtual machine based on a custom image available in the lab:


    az lab vm create --resource-group DtlResourceGroup --lab-name MyLab --name 'MyTestVm' --image "My Custom Image" --image-type custom --size 'Standard_D2s_v3' --admin-username 'AdminUser' --admin-password 'Password1!'

The **image-type** argument has changed from **gallery** to **custom**. The name of the image matches what you see if you were to create the virtual machine in the Azure portal.

The following command creates a VM from a marketplace image with ssh authentication:


    az lab vm create --lab-name sampleLabName --resource-group sampleLabResourceGroup --name sampleVMName --image "Ubuntu Server 16.04 LTS" --image-type gallery --size Standard_DS1_v2 --authentication-type  ssh --generate-ssh-keys --ip-configuration public 

You can also create virtual machines based on formulas by setting the **image-type** parameter to **formula**. If you need to choose a specific virtual network for your virtual machine, use the **vnet-name** and **subnet** parameters. For more information, see [az lab vm create](/en-us/cli/azure/lab/vm#az-lab-vm-create).

## Verify that the VM is available.[](#verify-that-the-vm-is-available)

Use the `az lab vm show` command to verify that the VM is available before you start and connect to it.

    az lab vm show --lab-name sampleLabName --name sampleVMName --resource-group sampleResourceGroup --expand 'properties($expand=ComputeVm,NetworkInterface)' --query '{status: computeVm.statuses[0].displayStatus, fqdn: fqdn, ipAddress: networkInterface.publicIpAddress}'

    {
      "fqdn": "lisalabvm.southcentralus.cloudapp.azure.com",
      "ipAddress": "13.85.228.112",
      "status": "Provisioning succeeded"
    }

## Start and connect to the virtual machine[](#start-and-connect-to-the-virtual-machine)

The following example command starts a VM:


    az lab vm start --lab-name sampleLabName --name sampleVMName --resource-group sampleLabResourceGroup

Connect to a VM: [SSH](../virtual-machines/linux/mac-create-ssh-keys) or [Remote Desktop](../virtual-machines/windows/connect-logon).


    ssh userName@ipAddressOrfqdn 

## Update the virtual machine[](#update-the-virtual-machine)

The following sample command applies artifacts to a VM:


    az lab vm apply-artifacts --lab-name  sampleLabName --name sampleVMName  --resource-group sampleResourceGroup  --artifacts @/artifacts.json


    [
      {
        "artifactId": "/artifactSources/public repo/artifacts/linux-java",
        "parameters": []
      },
      {
        "artifactId": "/artifactSources/public repo/artifacts/linux-install-nodejs",
        "parameters": []
      },
      {
        "artifactId": "/artifactSources/public repo/artifacts/linux-apt-package",
        "parameters": [
          {
            "name": "packages",
            "value": "abcd"
          },
          {
            "name": "update",
            "value": "true"
          },
          {
            "name": "options",
            "value": ""
          }
        ]
      } 
    ]

List artifacts available in the lab.


    az lab vm show --lab-name sampleLabName --name sampleVMName --resource-group sampleResourceGroup --expand "properties(\$expand=artifacts)" --query 'artifacts[].{artifactId: artifactId, status: status}'

<div class="codeHeader" id="code-try-11" data-bi-name="code-header"><span class="language">JSON</span> <button class="action" data-bi-name="copy" aria-label="Copy code"><span class="docon docon-edit-copy" role="presentation"></span><span>Copy</span></button></div>

    {
      "artifactId": "/subscriptions/abcdeftgh1213123/resourceGroups/lisalab123RG822645/providers/Microsoft.DevTestLab/labs/lisalab123/artifactSources/public repo/artifacts/linux-install-nodejs",
      "status": "Succeeded"
    }

## Stop and delete the virtual machine[](#stop-and-delete-the-virtual-machine)

The following sample command stops a VM.


    az lab vm stop --lab-name sampleLabName --name sampleVMName --resource-group sampleResourceGroup

Delete a VM.


    az lab vm delete --lab-name sampleLabName --name sampleVMName --resource-group sampleResourceGroup


