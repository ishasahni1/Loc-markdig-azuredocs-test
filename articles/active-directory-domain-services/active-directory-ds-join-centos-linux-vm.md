---
title: 'Azure Active Directory Domain Services: Join a CentOS VM to a managed domain | Microsoft Docs'
description: Join a CentOS Linux virtual machine to Azure AD Domain Services
services: active-directory-ds
documentationcenter: ''
author: mahesh-unnikrishnan
manager: mtillman
editor: curtand

ms.assetid: 16100caa-f209-4cb0-86d3-9e218aeb51c6
ms.service: active-directory-ds
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 10/16/2017
ms.author: maheshu

---
# Join a CentOS Linux virtual machine to a managed domain
This article shows you how to join a CentOS Linux virtual machine in Azure to an Azure AD Domain Services managed domain.

## Before you begin
To perform the tasks listed in this article, you need:
1. A valid **Azure subscription**.
2. An **Azure AD directory** - either synchronized with an on-premises directory or a cloud-only directory.
3. **Azure AD Domain Services** must be enabled for the Azure AD directory. If you haven't done so, follow all the tasks outlined in the [Getting Started guide](active-directory-ds-getting-started.md).
4. Ensure that you have configured the IP addresses of the managed domain as the DNS servers for the virtual network. For more information, see [how to update DNS settings for the Azure virtual network](active-directory-ds-getting-started-dns.md)
5. Complete the steps required to [synchronize passwords to your Azure AD Domain Services managed domain](active-directory-ds-getting-started-password-sync.md).


## Provision a CentOS Linux virtual machine
Provision a CentOS virtual machine in Azure, using any of the following methods:
* [Azure portal](../virtual-machines/linux/quick-create-portal.md)
* [Azure CLI](../virtual-machines/linux/quick-create-cli.md)
* [Azure PowerShell](../virtual-machines/linux/quick-create-powershell.md)

> [!IMPORTANT]
> * Deploy the virtual machine into the **same virtual network in which you have enabled Azure AD Domain Services**.
> * Pick a **different subnet** than the one in which you have enabled Azure AD Domain Services.
>


## Connect remotely to the newly provisioned Linux virtual machine
The CentOS virtual machine has been provisioned in Azure. The next task is to connect remotely to the virtual machine using the local administrator account created while provisioning the VM.

Follow the instructions in the article [How to log on to a virtual machine running Linux](../virtual-machines/linux/mac-create-ssh-keys.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).


## Configure the hosts file on the Linux virtual machine
In your SSH terminal, edit the /etc/hosts file and update your machine’s IP address and hostname.

```
sudo vi /etc/hosts
```

In the hosts file, enter the following value:

```
127.0.0.1 contoso-centos.contoso100.com contoso-centos
```
Here, 'contoso100.com' is the DNS domain name of your managed domain. 'contoso-centos' is the hostname of the CentOS virtual machine you are joining to the managed domain.


## Install required packages on the Linux virtual machine
Next, install packages required for domain join on the virtual machine. In your SSH terminal, type the following command to install the required packages:

    ```
    sudo yum install realmd sssd krb5-workstation krb5-libs oddjob oddjob-mkhomedir samba-common-tools
    ```


## Join the Linux virtual machine to the managed domain
Now that the required packages are installed on the Linux virtual machine, the next task is to join the virtual machine to the managed domain.

1. Discover the AAD Domain Services managed domain. In your SSH terminal, type the following command:

    ```
    sudo realm discover CONTOSO100.COM
    ```

   > [!NOTE]
   > **Troubleshooting:**
   > If *realm discover* is unable to find your managed domain:  
   >    * Ensure that the domain is reachable from the virtual machine (try ping).  
   >    * Check that the virtual machine has indeed been deployed to the same virtual network in which the managed domain is available. 
   >    * Check to see if you have updated the DNS server settings for the virtual network to point to the domain controllers of the managed domain.  

2. Initialize Kerberos. In your SSH terminal, type the following command:

    > [!TIP]
    > * Specify a user who belongs to the 'AAD DC Administrators' group.
    > * Specify the domain name in capital letters, else kinit fails.
    >

    ```
    kinit bob@CONTOSO100.COM
    ```

3. Join the machine to the domain. In your SSH terminal, type the following command:

    > [!TIP]
    > Use the same user account you specified in the preceding step ('kinit').
    >

    ```
    sudo realm join --verbose CONTOSO100.COM -U 'bob@CONTOSO100.COM'
    ```

You should get a message ("Successfully enrolled machine in realm") when the machine is successfully joined to the managed domain.


## Verify domain join
Verify whether the machine has been successfully joined to the managed domain. Connect to the domain joined CentOS VM using a different SSH connection. Use a domain user account and then check to see if the user account is resolved correctly.

1. In your SSH terminal, type the following command to connect to the domain joined CentOS virtual machine using SSH. Use a domain account that belongs to the managed domain (for example, 'bob@CONTOSO100.COM' in this case.)
    ```
    ssh -l bob@CONTOSO100.COM contoso-centos.contoso100.com
    ```

2. In your SSH terminal, type the following command to see if the home directory was initialized correctly.
    ```
    pwd
    ```

3. In your SSH terminal, type the following command to see if the group memberships are being resolved correctly.
    ```
    id
    ```


## Troubleshooting domain join
Refer to the [Troubleshooting domain join](active-directory-ds-admin-guide-join-windows-vm-portal.md#troubleshooting-domain-join) article.

## Related Content
* [Azure AD Domain Services - Getting Started guide](active-directory-ds-getting-started.md)
* [Join a Windows Server virtual machine to an Azure AD Domain Services managed domain](active-directory-ds-admin-guide-join-windows-vm.md)
* [How to log on to a virtual machine running Linux](../virtual-machines/linux/mac-create-ssh-keys.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).
* [Installing Kerberos](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Managing_Smart_Cards/installing-kerberos.html)
* [Red Hat Enterprise Linux 7 - Windows Integration Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Windows_Integration_Guide/index.html)
