---
title: Create an application gateway with SSL termination - Azure portal | Microsoft Docs
description: Learn how to create an application gateway and add a certificate for SSL termination using the Azure portal.
services: application-gateway
author: davidmu1
manager: timlt
editor: tysonn
tags: azure-resource-manager

ms.service: application-gateway
ms.topic: article
ms.workload: infrastructure-services
ms.date: 01/26/2018
ms.author: davidmu

---
# Create an application gateway with SSL termination using the Azure portal

You can use the Azure portal to create an [application gateway](application-gateway-introduction.md) with a certificate for SSL termination that uses virtual machines for backend servers.

In this article, you learn how to:

> [!div class="checklist"]
> * Create a self-signed certificate
> * Create an application gateway with the certificate
> * Create the virtual machines used as backend servers

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

## Log in to Azure

Log in to the Azure portal at [http://portal.azure.com](http://portal.azure.com)

## Create a self-signed certificate

In this section, you use [New-SelfSignedCertificate](https://docs.microsoft.com/powershell/module/pkiclient/new-selfsignedcertificate) to create a self-signed certificate that you upload to the Azure portal when you create the listener for the application gateway.

On your local computer, open a Windows PowerShell window as an administrator. Run the following command to create the certificate:

```powershell
New-SelfSignedCertificate \
  -certstorelocation cert:\localmachine\my \
  -dnsname www.contoso.com
```

You should see something like this response:

```
PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\my

Thumbprint                                Subject
----------                                -------
E1E81C23B3AD33F9B4D1717B20AB65DBB91AC630  CN=www.contoso.com

Use [Export-PfxCertificate](https://docs.microsoft.com/powershell/module/pkiclient/export-pfxcertificate) with the Thumbprint that was returned to export a pfx file from the certificate:
```

```powershell
$pwd = ConvertTo-SecureString -String "Azure123456!" -Force -AsPlainText
Export-PfxCertificate \
  -cert cert:\localMachine\my\E1E81C23B3AD33F9B4D1717B20AB65DBB91AC630 \
  -FilePath c:\appgwcert.pfx \
  -Password $pwd
```

## Create an application gateway

A virtual network is needed for communication between the resources that you create. Two subnets are created in this example: one for the application gateway, and the other for the backend servers. You can create a virtual network at the same time that you create the application gateway.

1. Click **New** found on the upper left-hand corner of the Azure portal.
2. Select **Networking** and then select **Application Gateway** in the Featured list.
3. Enter *myAppGateway* for the name of the application gateway and *myResourceGroupAG* for the new resource group.
4. Accept the default values for the other settings and then click **OK**.
5. Click **Choose a virtual network**, click **Create new**, and then enter these values for the virtual network:

   - *myVNet* - for the name of the virtual network.
   - *10.0.0.0/16* - for the virtual network address space.
   - *myAGSubnet* - for the subnet name.
   - *10.0.0.0/24* - for the subnet address space.

     ![Create virtual network](./media/application-gateway-ssl-portal/application-gateway-vnet.png)

6. Click **OK** to create the virtual network and subnet.
7. Click **Choose a public IP address**, click **Create new**, and then enter the name of the public IP address. In this example, the public IP address is named *myAGPublicIPAddress*. Accept the default values for the other settings and then click **OK**.
8. Click **HTTPS** for the protocol of the listener and make sure that the port is defined as **443**.
9. Click the folder icon and browse to the *appgwcert.pfx* certificate that you previously created to upload it.
10. Enter *mycert1* for the name of the certificate and *Azure123456!* for the password, and then click **OK**.

    ![Create new application gateway](./media/application-gateway-ssl-portal/application-gateway-create.png)

11. Review the settings on the summary page, and then click **OK** to create the network resources and the application gateway. It may take several minutes for the application gateway to be created, wait until the deployment finishes successfully before moving on to the next section.

### Add a subnet

1. Click **All resources** in the left-hand menu, and then click **myVNet** from the resources list.
2. Click **Subnets**, and then click **Subnet**.

    ![Create subnet](./media/application-gateway-ssl-portal/application-gateway-subnet.png)

3. Enter *myBackendSubnet* for the name of the subnet and then click **OK**.

## Create backend servers

In this example, you create two virtual machines to be used as backend servers for the application gateway. You also install IIS on the virtual machines to verify that the application gateway was successfully created.

### Create a virtual machine

1. Click **New**.
2. Click **Compute** and then select **Windows Server 2016 Datacenter** in the Featured list.
3. Enter these values for the virtual machine:

    - *myVM* - for the name of the virtual machine.
    - *azureuser* - for the administrator user name.
    - *Azure123456!* for the password.
    - Select **Use existing**, and then select *myResourceGroupAG*.

4. Click **OK**.
5. Select **DS1_V2** for the size of the virtual machine, and click **Select**.
6. Make sure that **myVNet** is selected for the virtual network and the subnet is **myBackendSubnet**. 
7. Click **Disabled** to disable boot diagnostics.
8. Click **OK**, review the settings on the summary page, and then click **Create**.

### Install IIS

1. Open the interactive shell and make sure that it is set to **PowerShell**.

    ![Install custom extension](./media/application-gateway-ssl-portal/application-gateway-extension.png)

2. Run the following command to install IIS on the virtual machine: 

    ```azurepowershell-interactive
    Set-AzureRmVMExtension `
      -ResourceGroupName myResourceGroupAG `
      -ExtensionName IIS `
      -VMName myVM `
      -Publisher Microsoft.Compute `
      -ExtensionType CustomScriptExtension `
      -TypeHandlerVersion 1.4 `
      -SettingString '{"commandToExecute":"powershell Add-WindowsFeature Web-Server; powershell Add-Content -Path \"C:\\inetpub\\wwwroot\\Default.htm\" -Value $($env:computername)"}' `
      -Location EastUS
    ```

3. Create a second virtual machine and install IIS using the steps that you just finished. Enter *myVM2* for its name and for VMName in Set-AzureRmVMExtension.

### Add backend servers

3. Click **All resources**, and then click **myAppGateway**.
4. Click **Backend pools**. A default pool was automatically created with the application gateway. Click **appGateayBackendPool**.
5. Click **Add target** to add each virtual machine that you created to the backend pool.

    ![Add backend servers](./media/application-gateway-ssl-portal/application-gateway-backend.png)

6. Click **Save**.

## Test the application gateway

1. Click **All resources**, and then click **myAGPublicIPAddress**.

    ![Record application gateway public IP address](./media/application-gateway-ssl-portal/application-gateway-ag-address.png)

2. Copy the public IP address, and then paste it into the address bar of your browser. To accept the security warning if you used a self-signed certificate, select Details and then Go on to the webpage:

    ![Secure warning](./media/application-gateway-ssl-portal/application-gateway-secure.png)

    Your secured IIS website is then displayed as in the following example:

    ![Test base URL in application gateway](./media/application-gateway-ssl-portal/application-gateway-iistest.png)

## Next steps

In this tutorial, you learned how to:

> [!div class="checklist"]
> * Create a self-signed certificate
> * Create an application gateway with the certificate
> * Create the virtual machines used as backend servers

To learn more about application gateways and their associated resources, continue to the how-to articles.