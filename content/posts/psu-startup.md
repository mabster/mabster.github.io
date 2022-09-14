+++ 
draft = false
date = 2022-08-20T14:30:00+10:00
title = "PowerShell Universal Startup Script"
description = "Here's a script I made for PowerShell Universal to run on server startup."
slug = ""
authors = []
tags = ['PowerShell', 'Universal', 'Azure']
categories = []
externalLink = ""
series = []
+++

I've been playing with [PowerShell Universal](https://ironmansoftware.com/powershell-universal) for almost a year now, and I thought this would be a great place to share some tips, tricks and scripts that have helped me out. And what better place to start than the `initialize.ps1` script? That's the script that runs right before the PSU server starts.

The script lives in the `.universal` folder inside your Repository folder, so create it if it doesn't exist already.

<!--more--> 

This script will install a bunch of modules that we use, but it installs them outside of the Repository folder (in a folder called 'Modules' alongside it, in fact) so they are not synced to GitHub. That way the big binary modules like `Microsoft.Graph.Authentication` and `Az.Accounts` are not in your Git repo.

Since we host PSU in an Azure App Service, this script also connects to Azure as a managed identity and registers our KeyVault as the default place to store secrets.

Obviously I'm not including any personal information in this copy of the script, so make sure you swap in your subscription ID, KeyVault name etc!

```powershell
# Good chance the $Repository variable doesn't exist yet, so set it to the Repository folder just in case.
$repo = $Repository ?? 'D:\home\data\PowershellUniversal\Repository'

# Ensure that all required modules are installed outside of the repository
$modules = @(
  'Az.Accounts'
  'Az.KeyVault'
  'Microsoft.Graph.Authentication'
  'Microsoft.Graph.Users'
  # etc
)

# make a 'Modules' folder alongside the repository
$path = Join-Path $repo '..\Modules\'
if (-not (Test-Path $path)) {
  New-Item -Path $path -ItemType Directory -Force
}
$path = Resolve-Path $path

# Add it to the path
if ($env:PSModulePath -split ';' -notcontains $path) {
    $env:PSModulePath += ";$path"
}

# Save the modules to our new folder
$modules | Foreach-Object {
  try {
    $m = $_
    $installed = Get-PSResource -Name $m -Path $path
    if ($installed) {
      # Checking for updates 
      $mod = Find-PSResource -Name $m

      if ($mod.Version -le $installed.Version) {
        Write-Host "Skipping '$m'"
        return;
      }

      Write-Host "Uninstalling '$m' so we upgrade to version $($mod.Version)"
      Uninstall-PSResource -Name $m
    }

    Write-Host "Installing '$m'"
    Save-PSResource -Name $m -Path $path -TrustRepository -IncludeXML
  }
  catch {
    Write-Warning "Could not install/update module $m"
  }
}

# Now connect to Azure and register our KeyVault

$env:PSModulePath = ($env:PSModulePath -split ';' | Where-Object { $_ -notlike '*AzureResourceManager*' }) -join ';'

$sub = '<your Azure subscription ID>'
$vault = '<your Azure KeyVault name>'

Connect-AzAccount -Id -Scope CurrentUser 
Set-AzContext -SubscriptionId $sub 

Register-SecretVault -ModuleName Az.KeyVault -Name AzureKeyVault -VaultParameters @{ 
    AZKVaultName = $vault
    SubscriptionId = $sub
} -AllowClobber -DefaultVault
```
