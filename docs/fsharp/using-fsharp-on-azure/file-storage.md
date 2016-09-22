---
title: Get started with Azure File storage using F# 
description: Store file data in the cloud with Azure File storage, and mount your cloud file share from an Azure virtual machine (VM) or from an on-premises application running Windows.
keywords: visual f#, f#, functional programming, .NET, .NET Core, Azure
author: syclebsc
manager: jbronsk
ms.date: 09/20/2016
ms.topic: article
ms.prod: .net-core
ms.technology: .net-core-technologies
ms.devlang: dotnet
ms.assetid: 5c26a0aa-186e-476c-9f87-e0191754579e
---

# Get started with Azure File storage using F# 

Azure File storage is a service that offers file shares in the cloud using the standard [Server Message Block (SMB) Protocol](https://msdn.microsoft.com/library/windows/desktop/aa365233.aspx). Both SMB 2.1 and SMB 3.0 are supported. With Azure File storage, you can migrate legacy applications that rely on file shares to Azure quickly and without costly rewrites. Applications running in Azure virtual machines or cloud services or from on-premises clients can mount a file share in the cloud, just as a desktop application mounts a typical SMB share. Any number of application components can then mount and access the File storage share simultaneously.

### Conceptual overview

For a conceptual overview of file storage, please see [the .NET guide for file storage](https://azure.microsoft.com/en-us/documentation/articles/storage-dotnet-how-to-use-files/).

### Create an Azure storage account

To use this guide, you must first [create an Azure storage account](https://azure.microsoft.com/en-us/documentation/articles/storage-create-storage-account/).
You'll also need your storage access key for this account.

## Create an F# Script and Start F# Interactive

The samples in this article can be used in either an F# application or an F# script. To create an F# script, create a file with the `.fsx` extension, for example `blobs.fsx`, in your F# development environment.

Next, use a [package manager](package-management.md) such as Paket or NuGet to install the `WindowsAzure.Storage` package and reference `WindowsAzure.Storage.dll` in your script using a `#r` directive.

### Add namespace declarations

Add the following `open` statements to the top of the `blobs.fsx` file:

    open System
    open System.IO
    open Microsoft.Azure // Namespace for CloudConfigurationManager
    open Microsoft.WindowsAzure.Storage // Namespace for CloudStorageAccount
    open Microsoft.WindowsAzure.Storage.File // Namespace for File storage types

### Get your connection string

You'll need an Azure Storage connection string for this tutorial. For more information about connection strings, see [Configure Storage Connection Strings](https://azure.microsoft.com/en-us/documentation/articles/storage-configure-connection-string/).

For the tutorial, you'll enter your connection string in your script, like this:

    let storageConnString = "..." // fill this in from your storage account

However, this is a **bad idea** for real projects. Your storage account key is similar to the root password for your storage account. Always be careful to protect your storage account key. Avoid distributing it to other users, hard-coding it, or saving it in a plain-text file that is accessible to others. You can regenerate your key using the Azure Portal if you believe it may have been compromised.

For real applications, the best way to maintain your storage connection string is in a configuration file. To fetch the connection string from a configuration file, you can do this:

    // Parse the connection string and return a reference to the storage account.
    let storageConnString = 
        CloudConfigurationManager.GetSetting("StorageConnectionString")

Using Azure Configuration Manager is optional. You can also use an API such as the .NET Framework's `ConfigurationManager` type.

### Parse the connection string

To parse the connection string, use:

    // Parse the connection string and return a reference to the storage account.
    let storageAccount = CloudStorageAccount.Parse(storageConnString)

This will return a `CloudStorageAccount`.

### Create the File service client

The `CloudFileClient` type enables you to programmatically use files stored in File storage. Here's one way to create the service client:

    let fileClient = storageAccount.CreateCloudFileClient()

Now you are ready to write code that reads data from and writes data to Blob storage.

## Create a file share

This example shows how to create a file share if it does not already exist:

    let share = fileClient.GetShareReference("myFiles")
    share.CreateIfNotExists()

### Access the file share programmatically

Here, we get the root directory and get a sub-directory of the root. If the sub-directory exists, we get a file in the sub-directory, and if that exists too, we download the file, appending the contents to a local file.

    let rootDir = share.GetRootDirectoryReference()
    let subDir = rootDir.GetDirectoryReference("myLogs")

    if subDir.Exists() then
        let file = subDir.GetFileReference("log.txt")
        if file.Exists() then
            file.DownloadToFile("log.txt", FileMode.Append)

### Set the maximum size for a file share

The example below shows how to check the current usage for a share and how to set the quota for the share. `FetchAttributes` must be called to populate a share's `Properties`, and `SetProperties` to propagate local changes to Azure File storage.

    // stats.Usage is current usage in GB
    let stats = share.GetStats()
    share.FetchAttributes()

    // Set the quota to 10 GB plus current usage
    share.Properties.Quota <- stats.Usage + 10 |> Nullable
    share.SetProperties()

    // Remove the quota
    share.Properties.Quota <- Nullable()
    share.SetProperties()

### Generate a shared access signature for a file or file share

You can generate a shared access signature (SAS) for a file share or for an individual file. You can also create a shared access policy on a file share to manage shared access signatures. Creating a shared access policy is recommended, as it provides a means of revoking the SAS if it should be compromised.

Here, we create a shared access policy on a share, and then use that policy to provide the constraints for a SAS on a file in the share.

    // Create a 24 hour read/write policy.
    let policy = SharedAccessFilePolicy()
    policy.SharedAccessExpiryTime <- 
        DateTimeOffset.UtcNow.AddHours(24.) |> Nullable
    policy.Permissions <- 
        SharedAccessFilePermissions.Read ||| SharedAccessFilePermissions.Write

    // Set the policy on the share.
    let permissions = share.GetPermissions()
    permissions.SharedAccessPolicies.Add("policyName", policy)
    share.SetPermissions(permissions)

    let file = subDir.GetFileReference("log.txt")
    let sasToken = file.GetSharedAccessSignature(policy)
    let sasUri = Uri(file.StorageUri.PrimaryUri.ToString() + sasToken)

    let fileSas = CloudFile(sasUri)
    fileSas.UploadText("This write operation is authenticated via SAS")

For more information about creating and using shared access signatures, see [Using Shared Access Signatures (SAS)](https://azure.microsoft.com/en-gb/documentation/articles/storage-dotnet-shared-access-signature-part-1/) and [Create and use a SAS with Blob storage](https://azure.microsoft.com/en-gb/documentation/articles/storage-dotnet-shared-access-signature-part-2/).

### Copy files

You can copy a file to another file, a file to a blob, or a blob to a file. If you are copying a blob to a file, or a file to a blob, you *must* use a shared access signature (SAS) to authenticate the source object, even if you are copying within the same storage account.

### Copy a file to another file

Here, we copy a file to another file in the same share. Because this copy operation copies between files in the same storage account, you can use Shared Key authentication to perform the copy.

    let destFile = subDir.GetFileReference("log_copy.txt")
    destFile.StartCopy(file)

### Copy a file to a blob

Here, we create a file and copy it to a blob within the same storage account. We create a SAS for the source file, which the service uses to authenticate access to the source file during the copy operation.

    // Get a reference to the blob to which the file will be copied.
    let blobClient = storageAccount.CreateCloudBlobClient()
    let container = blobClient.GetContainerReference("myContainer")
    container.CreateIfNotExists()
    let destBlob = container.GetBlockBlobReference("log_blob.txt")

    let filePolicy = SharedAccessFilePolicy()
    filePolicy.Permissions <- SharedAccessFilePermissions.Read
    filePolicy.SharedAccessExpiryTime <- 
        DateTimeOffset.UtcNow.AddHours(24.) |> Nullable

    let fileSas2 = file.GetSharedAccessSignature(filePolicy)
    let sasUri2 = Uri(file.StorageUri.PrimaryUri.ToString() + fileSas2)
    destBlob.StartCopy(sasUri2)

You can copy a blob to a file in the same way. If the source object is a blob, then create a SAS to authenticate access to that blob during the copy operation.

## Troubleshooting File storage using metrics

Azure Storage Analytics supports metrics for File storage. With metrics data, you can trace requests and diagnose issues.

You can enable metrics for File storage from the [Azure Portal](https://portal.azure.com), or you can do it from F# like this:

    open Microsoft.WindowsAzure.Storage.File.Protocol
    open Microsoft.WindowsAzure.Storage.Shared.Protocol

    let props = FileServiceProperties()
    props.HourMetrics <- MetricsProperties()
    props.HourMetrics.MetricsLevel <- MetricsLevel.ServiceAndApi
    props.HourMetrics.RetentionDays <- 14 |> Nullable
    props.HourMetrics.Version <- "1.0"
    props.MinuteMetrics <- MetricsProperties()
    props.MinuteMetrics.MetricsLevel <- MetricsLevel.ServiceAndApi
    props.MinuteMetrics.RetentionDays <- 7 |> Nullable
    props.MinuteMetrics.Version <- "1.0"

    fileClient.SetServiceProperties(props)

## Next steps

See these links for more information about Azure File storage.

### Conceptual articles and videos

- [Azure Files Storage: a frictionless cloud SMB file system for Windows and Linux](https://azure.microsoft.com/documentation/videos/azurecon-2015-azure-files-storage-a-frictionless-cloud-smb-file-system-for-windows-and-linux/)
- [How to use Azure File Storage with Linux](https://azure.microsoft.com/en-gb/documentation/articles/storage-how-to-use-files-linux/)

### Tooling support for File storage

- [Using Azure PowerShell with Azure Storage](https://azure.microsoft.com/en-gb/documentation/articles/storage-powershell-guide-full/)
- [How to use AzCopy with Microsoft Azure Storage](https://azure.microsoft.com/en-gb/documentation/articles/storage-use-azcopy/)
- [Using the Azure CLI with Azure Storage](https://azure.microsoft.com/en-gb/documentation/articles/storage-azure-cli/#create-and-manage-file-shares)

### Reference

- [Storage Client Library for .NET reference](https://msdn.microsoft.com/library/azure/dn261237.aspx)
- [File Service REST API reference](http://msdn.microsoft.com/library/azure/dn167006.aspx)

### Blog posts

- [Azure File storage is now generally available](https://azure.microsoft.com/blog/azure-file-storage-now-generally-available/)
- [Inside Azure File Storage](https://azure.microsoft.com/blog/inside-azure-file-storage/) 
- [Introducing Microsoft Azure File Service](http://blogs.msdn.com/b/windowsazurestorage/archive/2014/05/12/introducing-microsoft-azure-file-service.aspx)
- [Persisting connections to Microsoft Azure Files](http://blogs.msdn.com/b/windowsazurestorage/archive/2014/05/27/persisting-connections-to-microsoft-azure-files.aspx)