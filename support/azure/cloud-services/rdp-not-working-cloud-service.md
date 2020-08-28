---
title: RDP not working for classic cloud service resource
description: Provides information about troubleshooting issues in which RDP is not working for Azure customers who are using classic cloud service resource.
ms.date: 06/22/2020
ms.prod-support-area-path: 
ms.reviewer: 
author: genlin
ms.author: genli
ms.service: cloud-services
---
# RDP not working for classic cloud service resource

This article provides information about troubleshooting issues in which RDP is not working for Azure customers who are using classic cloud service resource.

_Original product version:_ &nbsp; API Management Service  
_Original KB number:_ &nbsp; 4464927

## Symptoms

This article discusses one of the most common scenarios faced by the Azure customers who are using classic cloud service resource - RDP is not working with this error:

:::image type="content" source="media/rdp-not-working-cloud-service/4464925_en_1.png" alt-text="Screenshot of RDP error.":::

Sometimes you will not be able to download the RDP file itself and will get this error:

> "Failed to download the file. Error details: error 400 Bad Request".

## Cause

You may get the above errors for multiple reasons:

- RDP user account or Encryption certificate is expired.
- TCP Port 3389 for RDP is blocked.
- RDP extension is disabled.

## Resolution

Sometimes while troubleshooting the RDP issues, you might find that resetting the RDP user account expiration date or renewing the encryption certificate doesn't work. In those cases, follow these steps:

1. Create a self-signed certificate in .pfx format.
2. Upload the self-signed certificate in cloud service certificate store from the Azure portal.
3. Delete all the existing RDP extensions if present.
4. Re-enable Remote Desktop for the roles by using the self-signed certificate that you created in the step 1.

You can also perform the above four steps using a PowerShell script. Run the below PowerShell script in admin or elevated mode. Make sure you have [Azure PowerShell Service Management module](https://docs.microsoft.com/powershell/azure/servicemanagement/install-azure-ps?view=azuresmps-4.0.0)  installed in your system before running it.

```powershell
$SubscriptionId = "your-subscription-id" # Subscription Id$CloudServiceName = "mycloudservice" # Cloud Service name$CertPassword = "CertPassword" # Password for the self-signed certificate$CertExportFilePath = "C:\my-cert-file.pfx" # Local file path where self-signed certificate will be exported$RdpUserName = "RemoteUserName" # RDP user name$RdpUserPassw0rd = "RdpPassword" # RDP user password$Slot = "Production" # Cloud Service slot$RdpAccountExpiry = $(Get-Date).AddDays(365) # RDP user account expiration DateTime

# Creating self-signed certificateWrite-Host (Get-Date).ToString()" : Creating self-signed certificate." -ForegroundColor Magenta$cert = New-SelfSignedCertificate -DnsName ($CloudServiceName + ".cloudapp.net") -CertStoreLocation "cert:\LocalMachine\My" -KeyLength 2048 -KeySpec "KeyExchange"$SecureCertPassword = ConvertTo-SecureString -String $CertPassword -Force -AsPlainTextExport-PfxCertificate -Cert $cert -FilePath $CertExportFilePath -Password $SecureCertPasswordWrite-Host (Get-Date).ToString()" : Self-signed certificate created successfully at" $CertExportFilePath -ForegroundColor Magenta

# Login to your Azure accountWrite-Host (Get-Date).ToString()" : Logging into your Azure account." -ForegroundColor MagentaAdd-AzureAccountSelect-AzureSubscription -SubscriptionId $SubscriptionIdWrite-Host (Get-Date).ToString()" : Logged in successfully." -ForegroundColor Magenta

# Uploading self-signed certificate to the cloud service certificate storeWrite-Host (Get-Date).ToString()" : Uploading self-signed certificate to the cloud service certificate store." -ForegroundColor MagentaAdd-AzureCertificate -serviceName $CloudServiceName -certToDeploy $CertExportFilePath -password $CertPasswordWrite-Host (Get-Date).ToString()" : Self-signed certificate uploaded successfully." -ForegroundColor Magenta

# Delete all the existing RDP extensions for a given cloud service slotWrite-Host (Get-Date).ToString()" : Deleting all the existing RDP extensions for" $Slot "slot." -ForegroundColor MagentaRemove-AzureServiceRemoteDesktopExtension -ServiceName $CloudServiceName -UninstallConfiguration -Slot $SlotWrite-Host (Get-Date).ToString()" : Successfully deleted all the existing RDP extensions for" $Slot "slot." -ForegroundColor Magenta

# Enabling remote desktop extension on specified role(s) or all roles on a cloud service slotWrite-Host (Get-Date).ToString()" : Enabling remote desktop extension on all the roles." -ForegroundColor Magenta$SecureRdpPassword = ConvertTo-SecureString -String $RdpUserPassw0rd -Force -AsPlainText$Credential = New-Object System.Management.Automation.PSCredential $RdpUserName,$SecureRdpPasswordSet-AzureServiceRemoteDesktopExtension -ServiceName $CloudServiceName -Credential $Credential -CertificateThumbprint$cert.Thumbprint -Expiration $RdpAccountExpiry -Slot $SlotWrite-Host (Get-Date).ToString()" : Remote desktop extension applied successfully." -ForegroundColor Magenta
```

Output of the script will be something like below:

:::image type="content" source="media/rdp-not-working-cloud-service/4464926_en_1.png" alt-text="Screenshot of PowerShell outputs.":::

If you are not able to RDP after running the above script, then definitely it's a networking issue. There are few possible reasons:

- The network from where you are trying to RDP is blocking the traffic.
- There are some ACL rules configured in your cloud service.
- Firewall rules configured using startup tasks.
- If your cloud service is sitting behind an NSG, you may need to create rules that allow traffic on ports **3389** and **20000**. The RemoteForwarder and RemoteAccess agents require that port 20000* is open, which may be blocked if you have an NSG.

Most of time your corporate network blocks the RDP traffic due to security reasons. So the first thing you should check if you are able to reach to RDP ports 3389 and 20000 (if applicable as mentioned above) using [PsPing](https://docs.microsoft.com/sysinternals/downloads/psping)  or [PortQry](https://www.microsoft.com/download/details.aspx?id=24009) or [Telnet](https://blogs.technet.microsoft.com/danielmauser/2015/03/18/tip-installing-telnet-client-via-command-line/).

You can refer this [article](https://support.microsoft.com/help/4464850) where discusses how to troubleshoot RDP issues using various tools like PsPing and Network monitor. If you are not getting any response, try to RDP from a different network, may be home network or mobile hotspot, etc.