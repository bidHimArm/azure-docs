---
title: Template specs overview
description: Describes how to create template specs and share them with other users in your organization. 
ms.topic: conceptual
ms.date: 07/20/2020
ms.author: tomfitz
author: tfitzmac
---
# Azure Resource Manager template specs (Preview)

A template spec is a new resource type for storing an Azure Resource Manager template (ARM template) in Azure for later deployment. This resource type enables you to share ARM templates with other users in your organization. Just like any other Azure resource, you can use role-based access control (RBAC) to share the template spec. Users only need read access to the template spec to deploy its template, so you can share the template without allowing others to modify it.

**Microsoft.Resources/templateSpecs** is the new resource type for template specs. It consists of a main template and any number of linked templates. Azure securely stores template specs in resource groups. Template Specs support [versioning](#versioning).

To deploy the template spec, you use standard Azure tools like PowerShell, Azure CLI, Azure portal, REST, and other supported SDKs and clients. You use the same commands and pass in the same parameters for the template.

The benefit of using template specs is that teams in your organization don't have to recreate or copy templates for common scenarios. You create canonical templates and share them. The templates you include in a template spec should be verified by administrators in your organization to follow the organization's requirements and guidance.

> [!NOTE]
> Template Specs is currently in preview. To use it, you must [sign up for the wait list](https://aka.ms/templateSpecOnboarding).

## Create template spec

The following example shows a simple template for creating a storage account in Azure.

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "name": "[concat('storage', uniqueString(resourceGroup().id))]",
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-06-01",
      "location": "[resourceGroup().location]",
      "kind": "StorageV2",
      "sku": {
        "name": "Premium_LRS",
        "tier": "Premium"
      }
    }
  ]
}
```

When you create the template spec, the PowerShell or CLI commands are passed the main template file. If the main template references linked templates, the commands will find and package them to create the template spec. To learn more, see [Create a template spec with linked templates](#create-a-template-spec-with-linked-templates).

Create a template spec by using:

```azurepowershell
New-AzTemplateSpec -Name storageSpec -Version 1.0 -ResourceGroupName templateSpecsRg -TemplateJsonFile ./mainTemplate.json
```

You can view all template specs in your subscription by using:

```azurepowershell
Get-AzTemplateSpec
```

You can view details of a template spec, including its versions with:

```azurepowershell
Get-AzTemplateSpec -ResourceGroupName templateSpecsRG -Name storageSpec
```

## Deploy template spec

After you've created the template spec, users with **read** access to the template spec can deploy it. For information about granting access, see [Tutorial: Grant a group access to Azure resources using Azure PowerShell](../../role-based-access-control/tutorial-role-assignments-group-powershell.md).

Template specs can be deployed through the portal, PowerShell, Azure CLI, or as a linked template in a larger template deployment. Users in an organization can deploy a template spec to any scope in Azure (resource group, subscription, management group, or tenant).

Instead of passing in a path or URI for a template, you deploy a template spec by providing its resource ID. The resource ID has the following format:

**/subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Resources/templateSpecs/{template-spec-name}/versions/{template-spec-version}**

Notice that the resource ID includes a version number for the template spec.

For example, you deploy a template spec with the following PowerShell command.

```azurepowershell
$id = "/subscriptions/11111111-1111-1111-1111-111111111111/resourceGroups/templateSpecsRG/providers/Microsoft.Resources/templateSpecs/storageSpec/versions/1.0"

New-AzResourceGroupDeployment `
  -TemplateSpecId $id `
  -ResourceGroupName demoRG
```

In practice, you'll typically run `Get-AzTemplateSpec` to get the ID of the template spec you want to deploy.

```azurepowershell
$id = (Get-AzTemplateSpec -Name storageSpec -ResourceGroupName templateSpecsRg -Version 1.0).Version.Id

New-AzResourceGroupDeployment `
  -TemplateSpecId $id `
  -ResourceGroupName demoRG
```

## Create a template spec with linked templates

If the main template for your template spec references linked templates, the PowerShell and CLI commands can automatically find and package the linked templates from your local drive. You don't need to manually configure storage accounts or repositories to host the template specs - everything is self-contained in the template spec resource.

The following example consists of a main template with two linked templates. The example is only an excerpt of the template. Notice that it uses a property named `relativePath` to link to the other templates. You must use `apiVersion` of `2020-06-01` or later for the deployments resource.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    ...
    "resources": [
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2020-06-01",
            ...
            "properties": {
                "mode": "Incremental",
                "templateLink": {
                    "relativePath": "artifacts/webapp.json"
                }
            }
        },
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2020-06-01",
            ...
            "properties": {
                "mode": "Incremental",
                "templateLink": {
                    "relativePath": "artifacts/database.json"
                }
            }
        }
    ],
    "outputs": {}
}

```

When the PowerShell or CLI command to create the template spec is executed for the preceding example, the command finds three files - the main template, the web app template (`webapp.json`), and the database template (`database.json`) - and packages them into the template spec.

For more information, see [Tutorial: Create a template spec with linked templates](template-specs-create-linked.md).

## Deploy template spec as a linked template

Once you've created a template spec, it's easy to reuse it from an ARM template or another template spec. You link to a template spec by adding its resource ID to your template. The linked template spec is automatically deployed when you deploy the main template. This behavior lets you develop modular template specs, and reuse them as needed.

For example, you can create a template spec that deploys networking resources, and another template spec that deploys storage resources. In ARM templates, you link to these two template specs anytime you need to configure networking or storage resources.

The following example is similar to the earlier example, but you use the `id` property to link to a template spec rather than the `relativePath` property to link to a local template. Use `2020-06-01` for API version for the deployments resource. In the example, the template specs are in a resource group named **templateSpecsRG**.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    ...
    "resources": [
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2020-06-01",
            "name": "networkingDeployment",
            ...
            "properties": {
                "mode": "Incremental",
                "templateLink": {
                    "id": "[resourceId('templateSpecsRG', 'Microsoft.Resources/templateSpecs/versions', 'networkingSpec', '1.0')]"
                }
            }
        },
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2020-06-01",
            "name": "storageDeployment",
            ...
            "properties": {
                "mode": "Incremental",
                "templateLink": {
                    "id": "[resourceId('templateSpecsRG', 'Microsoft.Resources/templateSpecs/versions', 'storageSpec', '1.0')]"
                }
            }
        }
    ],
    "outputs": {}
}
```

For more information about linking template specs, see [Tutorial: Deploy a template spec as a linked template](template-specs-deploy-linked-template.md).

## Versioning

When you create a template spec, you provide a version number for it. As you iterate on the template code, you can either update an existing version (for hotfixes) or publish a new version. The version is a text string. You can choose to follow any versioning system, including semantic versioning. Users of the template spec can provide the version number they want to use when deploying it.

## Next steps

* To create and deploy a template spec, see [Quickstart: Create and deploy template spec](quickstart-create-template-specs.md).

* For more information about linking templates in template specs, see [Tutorial: Create a template spec with linked templates](template-specs-create-linked.md).

* For more information about deploying a template spec as a linked template, see [Tutorial: Deploy a template spec as a linked template](template-specs-deploy-linked-template.md).
