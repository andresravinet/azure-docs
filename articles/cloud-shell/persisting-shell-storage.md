---
author: sdwheeler
description: Walkthrough of how Azure Cloud Shell persists files.
manager: mkluck
ms.author: sewhee
ms.contributor: jahelmic
ms.date: 11/14/2022
ms.service: cloud-shell
ms.tgt_pltfrm: vm-linux
ms.topic: article
ms.workload: infrastructure-services
services: azure
tags: azure-resource-manager
title: Persist files in Azure Cloud Shell
---
# Persist files in Azure Cloud Shell

Cloud Shell uses Azure Files to persist files across sessions. On initial start, Cloud Shell prompts
you to associate a new or existing fileshare to persist files across sessions.

> [!NOTE]
> Bash and PowerShell share the same fileshare. Only one fileshare can be associated with
> automatic mounting in Cloud Shell.
>
> Azure storage firewall isn't supported for cloud shell storage accounts.

## Create new storage

When you use basic settings and select only a subscription, Cloud Shell creates three resources on
your behalf in the supported region that's nearest to you:

- Resource group: `cloud-shell-storage-<region>`
- Storage account: `cs<uniqueGuid>`
- fileshare: `cs-<user>-<domain>-com-<uniqueGuid>`

![Screenshot of choosing the subscription for your storage account][09]

The fileshare mounts as `clouddrive` in your `$HOME` directory. This is a one-time action, and the
fileshare mounts automatically in subsequent sessions.

The fileshare also contains a 5-GB image that automatically persists data in your `$HOME` directory.
This fileshare is used for both Bash and PowerShell.

## Use existing resources

Using the advanced option, you can associate existing resources. When selecting a Cloud Shell region,
you must select a backing storage account co-located in the same region. For example, if your
assigned region is West US then you must associate a fileshare that resides within West US as well.

When the storage setup prompt appears, select **Show advanced settings** to view more options. The
populated storage options filter for locally redundant storage (LRS), geo-redundant storage (GRS),
and zone-redundant storage (ZRS) accounts.

> [!NOTE]
> Using GRS or ZRS storage accounts are recommended for additional resiliency for your backing file
> share. Which type of redundancy depends on your goals and price preference.
> [Learn more about replication options for Azure Storage accounts][04].

![Screenshot of configuring your storage account][08]

## Securing storage access

For security, each user should create their own storage account. For Azure role-based access control
(Azure RBAC), users must have contributor access or above at the storage account level.

Cloud Shell uses an Azure fileshare in a storage account, inside a specified subscription. Due to
inherited permissions, users with sufficient access rights to the subscription can access all the
storage accounts, and file shares contained in the subscription.

Users should lock down access to their files by setting the permissions at the storage account or
the subscription level.

The Cloud Shell storage account contains files created by the Cloud Shell user in their home
directory, which may include sensitive information including access tokens or credentials.

## Supported storage regions

To find your current region you may run `env` in Bash and locate the variable `ACC_LOCATION`, or
from PowerShell run `$env:ACC_LOCATION`. File shares receive a 5-GB image created for you to persist
your `$HOME` directory.

Cloud Shell machines exist in the following regions:

|     Area     |               Region               |
| ------------ | ---------------------------------- |
| Americas     | East US, South Central US, West US |
| Europe       | North Europe, West Europe          |
| Asia Pacific | India Central, Southeast Asia      |

Customers should choose a primary region, unless they have a requirement that their data at rest be
stored in a particular region. If they have such a requirement, a secondary storage region should be
used.

### Secondary storage regions

If a secondary storage region is used, the associated Azure storage account resides in a different
region as the Cloud Shell machine that you're mounting them to. For example, you can set your
storage account to be located in Canada East, a secondary region, but your Cloud Shell machine is
still located in a primary region. Your data at rest is located in Canada, but it's processed in the
United States.

> [!NOTE]
> If a secondary region is used, file access and startup time for Cloud Shell may be slower.

A user can run `(Get-CloudDrive | Get-AzStorageAccount).Location` in PowerShell to see the location
of their fileshare.

## Restrict resource creation with an Azure resource policy

Storage accounts that you create in Cloud Shell are tagged with
`ms-resource-usage:azure-cloud-shell`. If you want to disallow users from creating storage accounts
in Cloud Shell, create an [Azure resource policy for tags][03] that is triggered by this specific
tag.

## How Cloud Shell storage works

Cloud Shell persists files through both of the following methods:

- Creating a disk image of your `$HOME` directory to persist all contents within the directory. The
  disk image is saved in your specified fileshare as `acc_<User>.img` at
  `fileshare.storage.windows.net/fileshare/.cloudconsole/acc_<User>.img`, and it automatically syncs
  changes.
- Mounting your specified fileshare as `clouddrive` in your `$HOME` directory for direct file-share
  interaction. `/Home/<User>/clouddrive` is mapped to `fileshare.storage.windows.net/fileshare`.

> [!NOTE]
> All files in your `$HOME` directory, such as SSH keys, are persisted in your user disk image,
> which is stored in your mounted fileshare. Apply best practices when you persist information in
> your `$HOME` directory and mounted fileshare.

## clouddrive commands

### Use the `clouddrive` command

In Cloud Shell, you can run a command called `clouddrive`, which enables you to manually update the
fileshare that's mounted to Cloud Shell.

![Screenshot of running the clouddrive command in bash][10]

### List `clouddrive`

To discover which fileshare is mounted as `clouddrive`, run the `df` command.

The file path to clouddrive shows your storage account name and fileshare in the URL. For example,
`//storageaccountname.file.core.windows.net/filesharename`

```bash
justin@Azure:~$ df
Filesystem                                          1K-blocks   Used  Available Use% Mounted on
overlay                                             29711408 5577940   24117084  19% /
tmpfs                                                 986716       0     986716   0% /dev
tmpfs                                                 986716       0     986716   0% /sys/fs/cgroup
/dev/sda1                                           29711408 5577940   24117084  19% /etc/hosts
shm                                                    65536       0      65536   0% /dev/shm
//mystoragename.file.core.windows.net/fileshareName 5368709120    64 5368709056   1% /home/justin/clouddrive
```

### Mount a new clouddrive

#### Prerequisites for manual mounting

You can update the fileshare that's associated with Cloud Shell using the `clouddrive mount`
command.

If you mount an existing fileshare, the storage accounts must be located in your select Cloud Shell
region. Retrieve the location by running `env` and checking the `ACC_LOCATION`.

#### The `clouddrive mount` command

> [!NOTE]
> If you're mounting a new fileshare, a new user image is created for your `$HOME` directory. Your
> previous `$HOME` image is kept in your previous fileshare.

Run the `clouddrive mount` command with the following parameters:

```bash
clouddrive mount -s mySubscription -g myRG -n storageAccountName -f fileShareName
```

To view more details, run `clouddrive mount -h`, as shown here:

![Screenshot of running the clouddrive mount command in bash][11]

### Unmount clouddrive

You can unmount a fileshare that's mounted to Cloud Shell at any time. Since Cloud Shell requires a
mounted fileshare to be used, Cloud Shell prompts you to create and mount another fileshare on the
next session.

1. Run `clouddrive unmount`.
1. Acknowledge and confirm prompts.

The unmounted fileshare continues to exist until you manually delete it. After unmounting, Cloud
Shell no longer searches for this fileshare in subsequent sessions. To view more details, run
`clouddrive unmount -h`, as shown here:

![Screenshot of running the clouddrive unmount command in bash][12]

> [!WARNING]
> Although running this command doesn't delete any resources, manually deleting a resource group,
> storage account, or fileshare that's mapped to Cloud Shell erases your `$HOME` directory disk
> image and any files in your fileshare. This action can't be undone.

## PowerShell-specific commands

### List `clouddrive` Azure file shares

The `Get-CloudDrive` cmdlet retrieves the Azure fileshare information currently mounted by the
`clouddrive` in Cloud Shell.

![Screenshot of running the Get-CloudDrive command in PowerShell][07]

### Unmount `clouddrive`

You can unmount an Azure fileshare that's mounted to Cloud Shell at any time. The
`Dismount-CloudDrive` cmdlet unmounts an Azure fileshare from the current storage account.
Dismounting the `clouddrive` terminates the current session.

If the Azure fileshare has been removed, you'll be prompted to create and mount a new Azure
fileshare in the next session.

![Screenshot of running the Dismount-CloudDrive command in PowerShell][06]

## Transfer local files to Cloud Shell

The `clouddrive` directory syncs with the Azure portal storage blade. Use this blade to transfer
local files to or from your file share. Updating files from within Cloud Shell is reflected in the
file storage GUI when you refresh the blade.

### Download files

![Screenshot listing local files in the Azure portal][13]
1. In the Azure portal, go to the mounted file share.
2. Select the target file.
3. Select the **Download** button.

### Upload files

![Screenshot showing how to upload files in the Azure portal][14]
1. Go to your mounted file share.
2. Select the **Upload** button.
3. Select the file or files that you want to upload.
4. Confirm the upload.

You should now see the files that are accessible in your `clouddrive` directory in Cloud Shell.

> [!NOTE]
> If you need to define a function in a file and call it from the PowerShell cmdlets, then the
> dot operator must be included. For example: `. .\MyFunctions.ps1`

## Next steps

- [Cloud Shell Quickstart][15]
- [Learn about Microsoft Azure Files storage][05]
- [Learn about storage tags][02]

<!-- link references -->
[01]: includes/cloud-shell-persisting-shell-storage-endblock.md
[02]: ../azure-resource-manager/management/tag-resources.md
[03]: ../governance/policy/samples/index.md
[04]: ../storage/common/storage-redundancy.md
[05]: ../storage/files/storage-files-introduction.md
[06]: media/persisting-shell-storage/dismount-clouddrive.png
[07]: media/persisting-shell-storage/get-clouddrive.png
[08]: media/persisting-shell-storage/advanced-storage.png
[09]: media/persisting-shell-storage/basic-storage.png
[10]: media/persisting-shell-storage/clouddrive-h.png
[11]: media/persisting-shell-storage/mount-h.png
[12]: media/persisting-shell-storage/unmount-h.png
[13]: media/persisting-shell-storage/download.png
[14]: media/persisting-shell-storage/upload.png
[15]: quickstart.md
