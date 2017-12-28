---
title: "使用 Azure CLI 管理 Azure 磁碟 | Microsoft Docs"
description: "教學課程 - 使用 Azure CLI 管理 Azure 磁碟"
services: virtual-machines-linux
documentationcenter: virtual-machines
author: neilpeterson
manager: timlt
editor: tysonn
tags: azure-service-management
ms.assetid: 
ms.service: virtual-machines-linux
ms.devlang: na
ms.topic: tutorial
ms.tgt_pltfrm: vm-linux
ms.workload: infrastructure
ms.date: 05/02/2017
ms.author: nepeters
ms.custom: mvc
ms.openlocfilehash: 5a7a58d4c402bcaf639bd255bb7c8b111694e548
ms.sourcegitcommit: c7215d71e1cdeab731dd923a9b6b6643cee6eb04
ms.translationtype: HT
ms.contentlocale: zh-TW
ms.lasthandoff: 11/17/2017
---
# <a name="manage-azure-disks-with-the-azure-cli"></a>使用 Azure CLI 管理 Azure 磁碟

Azure 虛擬機器使用硬碟來儲存 VM 作業系統、應用程式和資料。 建立 VM 時，請務必選擇適合所預期工作負載的磁碟大小和組態。 本教學課程涵蓋部署和管理 VM 磁碟。 您將了解：

> [!div class="checklist"]
> * OS 磁碟和暫存磁碟
> * 資料磁碟
> * 標準和進階磁碟
> * 磁碟效能
> * 連結及準備資料磁碟
> * 調整磁碟的大小
> * 磁碟快照集


[!INCLUDE [cloud-shell-try-it.md](../../../includes/cloud-shell-try-it.md)]

如果您選擇在本機安裝和使用 CLI，本教學課程會要求您執行 Azure CLI 2.0.4 版或更新版本。 執行 `az --version` 以尋找版本。 如果您需要安裝或升級，請參閱[安裝 Azure CLI 2.0]( /cli/azure/install-azure-cli)。 

## <a name="default-azure-disks"></a>預設 Azure 磁碟

建立 Azure 虛擬機器後，有兩個磁碟會自動連結到虛擬機器。 

**作業系統磁碟** - 作業系統磁碟可裝載 VM 作業系統，其大小可以高達 1 TB。 OS 磁碟預設會標示為 /dev/sda。 OS 磁碟的磁碟快取組態已針對 OS 效能進行最佳化。 因此OS 磁碟**不得**裝載應用程式或資料。 請使用其他磁碟存檔資料，本文稍後會詳細說明。 

**暫存磁碟** - 暫存磁碟會使用與 VM 位於相同 Azure 主機的固態磁碟機。 暫存磁碟的效能非常好，可用於暫存資料處理等作業。 不過，如果 VM 移至新的主機，則會移除儲存在暫存磁碟上的任何資料。 暫存磁碟的大小取決於 VM 大小。 暫存磁碟會標示為 /dev/sdb，其掛接點為 /mnt。

### <a name="temporary-disk-sizes"></a>暫存磁碟大小

| 類型 | VM 大小 | 暫存磁碟大小上限 (GB) |
|----|----|----|
| [一般用途](sizes-general.md) | A 和 D 系列 | 800 |
| [計算最佳化](sizes-compute.md) | F 系列 | 800 |
| [記憶體最佳化](../virtual-machines-windows-sizes-memory.md) | D 和 G 系列 | 6144 |
| [儲存體最佳化](../virtual-machines-windows-sizes-storage.md) | L 系列 | 5630 |
| [GPU](sizes-gpu.md) | N 系列 | 1440 |
| [高效能](sizes-hpc.md) | A 和 H 系列 | 2000 |

## <a name="azure-data-disks"></a>Azure 資料磁碟

您可以新增額外資料磁碟，以便安裝應用程式和儲存資料。 資料磁碟應用來存儲日後需要使用的數據資料。 每個資料磁碟的最大容量為 1 TB。 虛擬機器的大小會決定可連結到 VM 的資料磁碟數目。 每個 VM vCPU 可以連結兩個資料磁碟。 

### <a name="max-data-disks-per-vm"></a>每部 VM 的資料磁碟上限

| 類型 | VM 大小 | 每部 VM 的資料磁碟上限 |
|----|----|----|
| [一般用途](sizes-general.md) | A 和 D 系列 | 32 |
| [計算最佳化](sizes-compute.md) | F 系列 | 32 |
| [記憶體最佳化](../virtual-machines-windows-sizes-memory.md) | D 和 G 系列 | 64 |
| [儲存體最佳化](../virtual-machines-windows-sizes-storage.md) | L 系列 | 64 |
| [GPU](sizes-gpu.md) | N 系列 | 48 |
| [高效能](sizes-hpc.md) | A 和 H 系列 | 32 |

## <a name="vm-disk-types"></a>VM 磁碟類型

Azure 提供兩種類型的磁碟。

### <a name="standard-disk"></a>標準磁碟

標準儲存體是以 HDD 為後盾，既可提供符合成本效益的儲存體，又可保持相當效能。 標準磁碟適合用於具成本效益的開發和測試工作負載。

### <a name="premium-disk"></a>進階磁碟

進階磁碟是以 SSD 為基礎的高效能、低延遲磁碟為藍本。 最適合用於執行生產工作負載的 VM。 進階儲存體支援 DS 系列、DSv2 系列、GS 系列和 FS 系列 VM。 進階磁碟有三種類型 (P10、P20、P30)，磁碟的大小可決定磁碟類型。 進行選取時，磁碟大小值會上調為下一個類型。 例如，如果磁碟大小少於 128 GB，則磁碟類型為 P10。 如果磁碟大小介於 129 GB 與 512 GB 之間，則大小為 P20。 任何超過 512 GB 的磁碟，其大小是 P30。

### <a name="premium-disk-performance"></a>進階磁碟效能

|進階儲存體磁碟類型 | P10 | P20 | P30 |
| --- | --- | --- | --- |
| 磁碟大小 (上調) | 128 GB | 512 GB | 1,024 GB (1 TB) |
| 每一磁碟的 IOPS 上限 | 500 | 2,300 | 5,000 |
每一磁碟的輸送量 | 100 MB/秒 | 150 MB/秒 | 200 MB/秒 |

雖然上表指出每個磁碟的最大 IOPS，但可藉由分割多個資料磁碟來達到較高等級的效能。 例如，Standard_GS5 VM 最高可達到 80,000 IOPS。 如需每部 VM 的最大 IOPS 詳細資訊，請參閱 [Linux VM 大小](sizes.md)。

## <a name="create-and-attach-disks"></a>建立和連結磁碟

您可以建立資料磁碟並在建立 VM 時連結，或連結至現有的 VM。

### <a name="attach-disk-at-vm-creation"></a>在建立 VM 時連結磁碟

使用 [az group create](https://docs.microsoft.com/cli/azure/group#az_group_create) 命令來建立資源群組。 

```azurecli-interactive 
az group create --name myResourceGroupDisk --location eastus
```

使用 [az vm create]( /cli/azure/vm#create) 命令來建立 VM。 `--datadisk-sizes-gb` 引數用來指定應該建立一個額外的磁碟並連結至虛擬機器。 若要建立並連結多個磁碟，請使用以空格分隔的磁碟大小值清單。 在下列範例中，會建立具有兩個資料磁碟 (均為 128 GB) 的 VM。 因為磁碟大小是 128 GB，所以這些磁碟都會設為 P10，其可提供每個磁碟最高 500 IOPS。

```azurecli-interactive 
az vm create \
  --resource-group myResourceGroupDisk \
  --name myVM \
  --image UbuntuLTS \
  --size Standard_DS2_v2 \
  --data-disk-sizes-gb 128 128 \
  --generate-ssh-keys
```

### <a name="attach-disk-to-existing-vm"></a>將磁碟連結至現有的 VM

若要建立新的磁碟並將它連結至現有的虛擬機器，請使用 [az vm disk attach](/cli/azure/vm/disk#attach) 命令。 下列範例會建立進階磁碟 (大小為 128 GB)，並將它連結至最後一個步驟中建立的 VM。

```azurecli-interactive 
az vm disk attach --vm-name myVM --resource-group myResourceGroupDisk --disk myDataDisk --size-gb 128 --sku Premium_LRS --new 
```

## <a name="prepare-data-disks"></a>準備資料磁碟

將磁碟連結到虛擬機器後，必須將作業系統設定為使用該磁碟。 下列範例示範如何手動設定磁碟。 使用 cloud-init (涵蓋於[稍後的教學課程](./tutorial-automate-vm-deployment.md)中) 也可以將此程序自動化。

### <a name="manual-configuration"></a>手動設定

建立虛擬機器的 SSH 連線。 以虛擬機器的公用 IP 位址取代範例 IP 位址。

```azurecli-interactive 
ssh 52.174.34.95
```

使用 `fdisk` 分割磁碟。

```bash
(echo n; echo p; echo 1; echo ; echo ; echo w) | sudo fdisk /dev/sdc
```

使用 `mkfs` 命令將檔案系統寫入至磁碟分割。

```bash
sudo mkfs -t ext4 /dev/sdc1
```

掛接新磁碟，使其可在作業系統中存取。

```bash
sudo mkdir /datadrive && sudo mount /dev/sdc1 /datadrive
```

現在可以透過 datadrive 掛接點存取磁碟，而執行 `df -h` 命令可驗證此掛接點。 

```bash
df -h
```

輸出會顯示掛接在 /datadrive 上的新磁碟機。

```bash
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        30G  1.4G   28G   5% /
/dev/sdb1       6.8G   16M  6.4G   1% /mnt
/dev/sdc1        50G   52M   47G   1% /datadrive
```

為了確保磁碟機會在重新開機之後重新掛接，必須將磁碟機新增至 /etc/fstab 檔案。 若要這麼做，請使用 `blkid` 公用程式取得磁碟的 UUID。

```bash
sudo -i blkid
```

輸出會顯示磁碟機的 UUID，在此情況下為 `/dev/sdc1`。

```bash
/dev/sdc1: UUID="33333333-3b3b-3c3c-3d3d-3e3e3e3e3e3e" TYPE="ext4"
```

將類似以下的一行新增至 /etc/fstab 檔案。 也請注意，使用 barrier=0 可以停用寫入屏障，此組態可以改善磁碟效能。 

```bash
UUID=33333333-3b3b-3c3c-3d3d-3e3e3e3e3e3e   /datadrive  ext4    defaults,nofail,barrier=0   1  2
```

現在已設定磁碟，請關閉 SSH 工作階段。

```bash
exit
```

## <a name="resize-vm-disk"></a>調整 VM 磁碟的大小

部署 VM 後，作業系統磁碟或任何連結的資料磁碟可以增加大小。 需要更多儲存空間或更高層級的效能 (P10、P20、P30) 時，增加磁碟的大小很有幫助。 請注意，磁碟的大小不能減少。

增加磁碟大小之前，需要有磁碟的識別碼或名稱。 使用 [az disk list](/cli/azure/disk#az_disk_list) 命令來傳回資源群組中的所有磁碟。 記下您想要調整大小的磁碟名稱。

```azurecli-interactive 
az disk list -g myResourceGroupDisk --query '[*].{Name:name,Gb:diskSizeGb,Tier:accountType}' --output table
```

此外，也必須將 VM 解除配置。 使用 [az vm deallocate]( /cli/azure/vm#deallocate) 命令來停止並解除配置 VM。

```azurecli-interactive 
az vm deallocate --resource-group myResourceGroupDisk --name myVM
```

使用 [az disk update](/cli/azure/vm/disk#update) 命令來調整磁碟的大小。 這個範例會將名為 myDataDisk 的磁碟大小調整為 1 TB。

```azurecli-interactive 
az disk update --name myDataDisk --resource-group myResourceGroupDisk --size-gb 1023
```

調整大小作業完成後，請啟動 VM。

```azurecli-interactive 
az vm start --resource-group myResourceGroupDisk --name myVM
```

如果您已調整作業系統磁碟的大小，則會自動擴充資料分割。 如果您已調整資料磁碟的大小，則必須在 VM 作業系統中擴充所有目前的資料分割。

## <a name="snapshot-azure-disks"></a>建立 Azure 磁碟快照集

建立磁碟快照集，以建立磁碟的讀取、時間點複本。 進行組態變更之前，Azure VM 快照集可用於快速儲存 VM 的狀態。 在證實不需要組態變更的事件中，可使用快照集還原 VM 狀態。 當 VM 有多個磁碟時，每個磁碟會各自產生快照集。 進行應用程式一致備份時，請考慮在建立磁碟快照集之前停止 VM。 或者，使用 [Azure 備份服務](/azure/backup/)，其可讓您在 VM 執行時執行自動化備份。

### <a name="create-snapshot"></a>建立快照集

建立虛擬機器磁碟快照集之前，需要磁碟的識別碼或名稱。 使用 [az vm show](https://docs.microsoft.com/en-us/cli/azure/vm#az_vm_show) 命令傳回磁碟識別碼。在此範例中，磁碟會儲存在變數中，以便使用於稍後的步驟。

```azurecli-interactive 
osdiskid=$(az vm show -g myResourceGroupDisk -n myVM --query "storageProfile.osDisk.managedDisk.id" -o tsv)
```

您現在有虛擬機器磁碟的識別碼，下列命令會建立磁碟的快照集。

```azurcli
az snapshot create -g myResourceGroupDisk --source "$osdiskid" --name osDisk-backup
```

### <a name="create-disk-from-snapshot"></a>從快照集建立磁碟

此快照集可以接著轉換成磁碟，進而用於重新建立虛擬機器。

```azurecli-interactive 
az disk create --resource-group myResourceGroupDisk --name mySnapshotDisk --source osDisk-backup
```

### <a name="restore-virtual-machine-from-snapshot"></a>從快照集還原虛擬機器

若要示範虛擬機器復原，請刪除現有的虛擬機器。 

```azurecli-interactive 
az vm delete --resource-group myResourceGroupDisk --name myVM
```

從快照磁碟建立新的虛擬機器。

```azurecli-interactive 
az vm create --resource-group myResourceGroupDisk --name myVM --attach-os-disk mySnapshotDisk --os-type linux
```

### <a name="reattach-data-disk"></a>重新連結資料磁碟

所有資料磁碟都必須重新連結至虛擬機器。

首先使用 [az disk list](https://docs.microsoft.com/cli/azure/disk#az_disk_list) 命令尋找資料磁碟名稱。 此範例會將磁碟名稱放入名為 datadisk 的變數，該變數使用於下一個步驟。

```azurecli-interactive 
datadisk=$(az disk list -g myResourceGroupDisk --query "[?contains(name,'myVM')].[name]" -o tsv)
```

使用 [az vm disk attach](https://docs.microsoft.com/cli/azure/vm/disk#az_vm_disk_attach) 命令來連結磁碟。

```azurecli-interactive 
az vm disk attach –g myResourceGroupDisk –-vm-name myVM –-disk $datadisk
```

## <a name="next-steps"></a>後續步驟

在本教學課程中，您已了解 VM 磁碟的相關主題，像是：

> [!div class="checklist"]
> * OS 磁碟和暫存磁碟
> * 資料磁碟
> * 標準和進階磁碟
> * 磁碟效能
> * 連結及準備資料磁碟
> * 調整磁碟的大小
> * 磁碟快照集

請前進到下一個教學課程，以了解如何自動設定 VM。

> [!div class="nextstepaction"]
> [自動設定 VM](./tutorial-automate-vm-deployment.md)
