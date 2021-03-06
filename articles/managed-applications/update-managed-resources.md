---
title: "更新 Azure 受管理應用程式中的資源 | Microsoft Docs"
description: "說明如何針對 Azure 受管理的應用程式使用受管理資源群組中的資源。"
services: managed-applications
author: tfitzmac
manager: timlt
ms.service: managed-applications
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.date: 10/26/2017
ms.author: tomfitz
ms.openlocfilehash: 59dce2fe7d91cc80f991e5ff298be7757ae19ef4
ms.sourcegitcommit: 3ab5ea589751d068d3e52db828742ce8ebed4761
ms.translationtype: HT
ms.contentlocale: zh-TW
ms.lasthandoff: 10/27/2017
---
# <a name="work-with-resources-in-the-managed-resource-group-for-azure-managed-application"></a>針對 Azure 受管理的應用程式使用受管理資源群組中的資源

本文說明如何更新部署在受管理應用程式中的資源。 身為受管理應用程式的發行者，您具有受管理資源群組中資源的存取權。 若要更新這些資源，您必須尋找與受管理應用程式相關聯的受管理資源群組，並且存取該資源群組中的資源。

本文假設您已在[受管理的 Web 應用程式 (IaaS) 與 Azure 管理服務](https://github.com/Azure/azure-managedapp-samples/tree/master/samples/201-managed-web-app)範例專案中部署受管理的應用程式。 受管理的應用程式包含 **Standard_D1_v2** 虛擬機器。 如果您尚未部署該受管理的應用程式，您仍然可以使用本文來熟悉更新受管理資源群組的步驟。

下圖顯示已部署的受管理應用程式。

![部署受管理的應用程式](./media/update-managed-resources/deployed.png)

您會在本文中使用 Azure CLI：

* 識別受管理的應用程式
* 識別受管理的資源群組
* 識別受管理資源群組中的虛擬機器資源
* 變更 VM 大小 (未使用時變更為較小大小，或者變更為較大大小以支援更多負載)
* 將指定允許位置的原則指派給受管理的資源群組

## <a name="get-managed-application-and-managed-resource-group"></a>取得受管理的應用程式和受管理的資源群組

若要取得資源群組中的受管理應用程式，請使用：

```azurecli-interactive
az managedapp list --query "[?contains(resourceGroup,'DemoApp')]"
```

若要取得受管理資源群組的識別碼，請使用：

```azurecli-interactive
az managedapp list --query "[?contains(resourceGroup,'DemoApp')].{ managedResourceGroup:managedResourceGroupId }"
```

## <a name="resize-vms-in-managed-resource-group"></a>調整受管理資源群組中 VM 的大小

若要查看受管理資源群組中的虛擬機器，請提供受管理資源群組的名稱。

```azurecli-interactive
az vm list -g DemoApp6zkevchqk7sfq --query "[].{VMName:name,OSType:storageProfile.osDisk.osType,VMSize:hardwareProfile.vmSize}"
```

若要更新 VM 大小，請使用：

```azurecli-interactive
az vm resize --size Standard_D2_v2 --ids $(az vm list -g DemoApp6zkevchqk7sfq --query "[].id" -o tsv)
```

在作業完成之後，請確認應用程式是在「標準 D2 v2」上執行。

![使用「標準 D2 v2」的受管理應用程式](./media/update-managed-resources/upgraded.png)

## <a name="apply-policy-to-managed-resource-group"></a>將原則套用至受管理的資源群組

取得受管理的資源群組並且在該範圍指派原則。 原則 **e56962a6-4747-49cd-b67b-bf8b01975c4c** 是內建原則，用來指定允許的位置。

```azurecli-interactive
managedGroup=$(az managedapp show --name <app-name> --resource-group DemoApp --query managedResourceGroupId --output tsv)

az policy assignment create --name locationAssignment --policy e56962a6-4747-49cd-b67b-bf8b01975c4c --scope $managedGroup --params '{
                            "listofallowedLocations": {
                                "value": [
                                    "northeurope",
                                    "westeurope"
                                ]
                            }
                        }'
```

若要查看允許的位置，請使用：

```azurecli-interactive
az policy assignment show --name locationAssignment --scope $managedGroup --query parameters.listofallowedLocations.value
```

原則指派會在入口網站顯示。

![檢視原則指派](./media/update-managed-resources/assignment.png)

## <a name="next-steps"></a>後續步驟

* 如需受管理應用程式的簡介，請參閱[受管理的應用程式概觀](overview.md)。
* 如需範例專案，請參閱[適用於 Azure 受管理的應用程式之範例專案](sample-projects.md)。
* 如需將受管理的應用程式發佈到 Azure Marketplace 的詳細資訊，請參閱 [Marketplace 中受管理的應用程式](publish-marketplace-app.md)。