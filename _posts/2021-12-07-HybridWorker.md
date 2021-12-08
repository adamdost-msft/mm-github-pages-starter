---
title: "Creating an Hybrid Worker in Azure"
date: 2021-12-01T18:23:30-04:00
categories:
  - blog
tags:
  - AzureAutomation
  - HybridWorkers
  - teamBlue
  - PowerShell
---

Using a Hybrid Worker with Azure Automation is something I became pretty familar with early in my career. They are great at dealing with changes to Active Directory or sending emails using an SMTP Server or even running a report on Windows Updates that will be pushed to your computers via Azure Automation Upgrade Management. However, the process of creating a Hybrid Worker has always been a confusing process for me and after recently struggling to understand the steps to creating one I wanted to share my experiences here.

You will need a few things before we get started:

- An Azure Automation Account with a Run As Connection created
- An Azure Virtual Machine you want to register as a Hybrid Worker that is registered with Log Analytics

# Gathering the required Information for setting up Hybrid Worker

Lets start by going to your Azure Automation Account you will be using. You will need to save a few pieces of information for later.

Navigate to Account Settings then _Keys_ on the left Navigation Menu save the following values

- Primary Access Key: You will use this to connect to your Azure Automation Account later when running the Hybrid Worker Script.
- URL: You will use this as a parameter when you create your Hybrid Worker later on.

# Creating the Hybrid Worker

Now that you have your information saved you can connect to the Azure Virtual Machine you will be turning into a Hybrid Worker Group. Lets connect to the Virtual Machine. Once connected open up an Admin Powershell Session and run the following:

```powershell
cd "C:\Program Files\Microsoft Monitoring Agent\Agent\AzureAutomation\<version>\HybridRegistration"
Import-Module .\HybridRegistration.psd1
```

This imports the required PowerShell module to enroll your Virtual Machine as a Hybrid Worker. Now that we have imported the module you can run the following command to create the Hybrid Worker.

```powershell
#Pass in your URL and Key you saved earlier then pass in a string to use as the name for the Hyrbid Worker Group in Azure.
Add-HybridRunbookWorker –GroupName <String> -Url <Url> -Key <String>
```

You have just created a Hybrid Worker that can be used to run scripts from Azure Automation Congratulations!

# Not So Fast!

While the Hybrid Worker has been created the Azure Run As Connection still must be on the Virtual Machine or it wont be able to authenicate with Azure. Thankfully Microsoft wrote us a script to do ll this magic for it. Copy the following script and **run it from Azure Automation not on the VM** When asked whether to run on Azure or a Hybrid Worker choose Hybrid Worker then select the Hybrid Worker Group you just created.

```powershell
<#PSScriptInfo
.VERSION 1.0
.GUID 3a796b9a-623d-499d-86c8-c249f10a6986
.AUTHOR Azure Automation Team
.COMPANYNAME Microsoft
.COPYRIGHT
.TAGS Azure Automation
.LICENSEURI
.PROJECTURI
.ICONURI
.EXTERNALMODULEDEPENDENCIES
.REQUIREDSCRIPTS
.EXTERNALSCRIPTDEPENDENCIES
.RELEASENOTES
#>

<#
.SYNOPSIS
Exports the Run As certificate from an Azure Automation account to a hybrid worker in that account.

.DESCRIPTION
This runbook exports the Run As certificate from an Azure Automation account to a hybrid worker in that account. Run this runbook on the hybrid worker where you want the certificate installed. This allows the use of the AzureRunAsConnection to authenticate to Azure and manage Azure resources from runbooks running on the hybrid worker.

.EXAMPLE
.\Export-RunAsCertificateToHybridWorker

.NOTES
LASTEDIT: 2016.10.13
#>

# Generate the password used for this certificate
Add-Type -AssemblyName System.Web -ErrorAction SilentlyContinue | Out-Null
$Password = [System.Web.Security.Membership]::GeneratePassword(25, 10)

# Stop on errors
$ErrorActionPreference = 'stop'

# Get the management certificate that will be used to make calls into Azure Service Management resources
$RunAsCert = Get-AutomationCertificate -Name "AzureRunAsCertificate"

# location to store temporary certificate in the Automation service host
$CertPath = Join-Path $env:temp  "AzureRunAsCertificate.pfx"

# Save the certificate
$Cert = $RunAsCert.Export("pfx",$Password)
Set-Content -Value $Cert -Path $CertPath -Force -Encoding Byte | Write-Verbose

Write-Output ("Importing certificate into $env:computername local machine root store from " + $CertPath)
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
Import-PfxCertificate -FilePath $CertPath -CertStoreLocation Cert:\LocalMachine\My -Password $SecurePassword | Write-Verbose

Remove-Item -Path $CertPath -ErrorAction SilentlyContinue | Out-Null

# Test to see if authentication to Azure Resource Manager is working
$RunAsConnection = Get-AutomationConnection -Name "AzureRunAsConnection"

Connect-AzAccount `
    -ServicePrincipal `
    -Tenant $RunAsConnection.TenantId `
    -ApplicationId $RunAsConnection.ApplicationId `
    -CertificateThumbprint $RunAsConnection.CertificateThumbprint | Write-Verbose

Set-AzContext -Subscription $RunAsConnection.SubscriptionID | Write-Verbose

# List automation accounts to confirm that Azure Resource Manager calls are working
Get-AzAutomationAccount | Select-Object AutomationAccountName
```

Once this script is executed successfully you will have a fully functional Hybrid Worker that can be used with Azure Automation.

# Why do I need an Azure Hybrid Worker?

Hybrid Workers can be useful if you are needing to automate changes in Active Directory which running a runbook in Azure wouldn't be able to acomplish for you. During my past experience I've used it to create/delete accounts as requests came in, Query Azure PaaS SQL Databases, and generate reports about Windows Updates in WSUS and then email the results using PowerShell.

# Wrap Up

I hope this blog post added a little more clarity with one of the easier ways to create a Hybrid Worker Group in Azure. Of course if you have any questions reach out to me on LinkedIn or email me at alex@alexdclark.dev.

-Alex
