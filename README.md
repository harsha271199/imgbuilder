# Azure Imgbuilder
![Alt Image-builder-overview](https://learn.microsoft.com/en-us/azure/virtual-machines/media/image-builder-overview/image-builder-flow.png)
## Description

Creating standardized virtual machine (VM) images allow organizations to migrate to the cloud and ensure consistency in the deployments. Users commonly want VMs to include predefined security and configuration settings as well as application software they own. However, setting up your own image build pipeline would require infrastructure and setup. With Azure VM Image Builder, you can take an ISO or Azure Marketplace image or existing Custom images and start creating your own golden images in a few steps.

The Azure VM Image Builder (AIB) lets you start with either a Windows or Linux-based Azure Marketplace VM, Red Hat Enterprise Linux (RHEL) ISO or your existing custom Images, and begin to add your own customizations.

## Feature Overview
The AIB is a fully managed Azure service that is accessible by an Azure first party resource provider.

The diagram below shows the end to end AIB pipeline, where you can see the three main components, source, customize and distribute, with their inputs and outputs.

Image Builder Overview

To describe the pipeline steps requires a configuration template, this is ingested into the service, then stored as an ‘ImageTemplate’ artifact. The template itself, is just a JSON template, and it can be nested into an ARM template, see examples [here](https://github.com/danielsollondon/azvmimagebuilder/tree/master/armTemplates ).

For the AIB to stand up the pipeline requires an invocation call into the AIB service referencing a stored template.


Create the Image Template, see the [quick quickstarts](https://github.com/danielsollondon/azvmimagebuilder/tree/master/quickquickstarts ) on how to do this.
Submit the Image Template, at this time AIB will download the RHEL ISO, and shell scripts needed, and store these in an automatically resource group created in the customers subscription, in this format : ‘IT_<DestinationResourceGroup>_<TemplateName>’. This will be removed when the Image Template artifact is deleted. You will also see the template artifact, that references these in the resource group referenced when creating the image template.
Invoking the AIB to create the image, will take the template artifact, then stand up a pipeline to create it, by standing up a VM, network, and storage in the automatically created resource group, ‘IT_<DestinationResourceGroup>_<TemplateName>’.

## Permissions Requirements
Image Builder uses Azure user-assigned managed identity to manage the necessary permissions.

Permissions Overview [public doc](https://learn.microsoft.com/en-us/azure/virtual-machines/image-builder-overview?bc=%2Fazure%2Fvirtual-machines%2Fwindows%2Fbreadcrumb%2Ftoc.json&tabs=azure-powershell#permissions )
Configure Image builder permissions with [CLI](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/image-builder-permissions-cli)  or [PowerShell](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/image-builder-permissions-powershell) 
Permission Requirements explained in detail on [GitHub](https://github.com/danielsollondon/azvmimagebuilder/blob/master/aibPermissions.md#azure-vm-image-builder-permissions-explained-and-requirements) 

## How it works
As mentioned above (TBD: Link to ‘Feature Overview), creating a VM image using AIB is a two-step process:

Creating an Image Template and submitted it to the AIB.
Invoking the AIB to create an Image from this template.




This article is to show you how you can create a basic customized image with these customizations:

* Installing a LOB App, [MS Teams](https://docs.microsoft.com/en-us/azure/virtual-desktop/teams-on-wvd)
* [Windows Restart](https://docs.microsoft.com/azure/virtual-machines/linux/image-builder-json?toc=%2Fazure%2Fvirtual-machines%2Fwindows%2Ftoc.json&bc=%2Fazure%2Fvirtual-machines%2Fwindows%2Fbreadcrumb%2Ftoc.json#windows-restart-customizer)
* [Windows Update](https://docs.microsoft.com/azure/virtual-machines/linux/image-builder-json?toc=%2Fazure%2Fvirtual-machines%2Fwindows%2Ftoc.json&bc=%2Fazure%2Fvirtual-machines%2Fwindows%2Fbreadcrumb%2Ftoc.json#windows-update-customizer)

We will show you how to automate this using the Azure VM Image Builder, and distibute to the Azure [Shared Image Gallery](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/shared-image-galleries), where you can replicate regions, control the scale, and share inside and outside your organizations.


To simplify deploying an AIB configuration template with PowerShell CLI, this example uses an Azure Resource Manager (ARM) template with the AIB template nested inside, and gives you other benefits for free, such as variables and parameter inputs etc. You can also pass parameters from the commandline too, which you will see here.

This walk through is intended to be a copy and paste exercise, and will provide you with a custom Win Server image (AIB also supports client images), showing you how you can easily create a custom image.

> Note! 
The scripts to install the apps are located in this repo, note, they are for illustration and testing ONLY, and **NOT** production. 

## Tips for Building Windows Images with AIB:
1. VM Size - When AIB runs, it uses a build VM to build the image, the default AIB size (Standard_D1_v2) is not suitable. Use Standard_D2_v2 or greater.
2. The example here uses the AIB [PowerShell customerizer scripts](https://docs.microsoft.com/azure/virtual-machines/linux/image-builder-json?toc=%2Fazure%2Fvirtual-machines%2Fwindows%2Ftoc.json&bc=%2Fazure%2Fvirtual-machines%2Fwindows%2Fbreadcrumb%2Ftoc.json#powershell-customizer), you will need to run these with the settings below. If you do not, the build will hang.
```text
  "runElevated": true,
  "runAsSystem": true,
```

3. Comment your code

The AIB build log (customization.log) is extremely verbose, if you comment your scripts using 'write-host' these will be sent to the logs, and make troubleshooting easier.

```PowerShell
 write-host 'AIB Customization: Starting OS Optimizations script'
```

4. Emit Exit Codes
AIB expects all scripts to return a 0 exit code, any non zero exit code will result in AIB failing the customization and stopping the build. If you have complex scripts, add instrumentation and emit exit codes, these will be shown in the customization.log.
```PowerShell
 Write-Host "Exit code: " $LASTEXITCODE
```
5. Test
Please test and test your code before on a standalone VM, ensure there are no user prompts, you are using the right privilege etc.

6. Networking: Set-NetAdapterAdvancedProperty 

This is being set in the optimization script, but fails the AIB build, as it disconnects the network, this is commented out. It is under investigation.

## PreReqs
You must have the latest Azure PowerShell CmdLets installed, see [here](https://docs.microsoft.com/en-us/powershell/azure/overview?view=azps-2.6.0) for install details.

```PowerShell
# Register for Azure Image Builder Feature
Register-AzProviderFeature -FeatureName VirtualMachineTemplatePreview -ProviderNamespace Microsoft.VirtualMachineImages

Get-AzProviderFeature -FeatureName VirtualMachineTemplatePreview -ProviderNamespace Microsoft.VirtualMachineImages
# wait until RegistrationState is set to 'Registered'

# check you are registered for the providers, ensure RegistrationState is set to 'Registered'.
Get-AzResourceProvider -ProviderNamespace Microsoft.VirtualMachineImages
Get-AzResourceProvider -ProviderNamespace Microsoft.Storage 
Get-AzResourceProvider -ProviderNamespace Microsoft.Compute
Get-AzResourceProvider -ProviderNamespace Microsoft.KeyVault

# If they do not saw registered, run the commented out code below.

## Register-AzResourceProvider -ProviderNamespace Microsoft.VirtualMachineImages
## Register-AzResourceProvider -ProviderNamespace Microsoft.Storage
## Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
## Register-AzResourceProvider -ProviderNamespace Microsoft.KeyVault
```

## Step 1: Set up environment and variables

```powerShell
# Step 1: Import module
Import-Module Az.Accounts

# Step 2: get existing context
$currentAzContext = Get-AzContext

# destination image resource group
$imageResourceGroup="img"

# location (see possible locations in main docs)
$location="eastus"

# your subscription, this will get your current subscription
$subscriptionID=$currentAzContext.Subscription.Id

# image template name
$imageTemplateName="win2019ImageTemplate01"

# distribution properties object name (runOutput), i.e. this gives you the properties of the managed image on completion
$runOutputName="sigOutput"

# create resource group
New-AzResourceGroup -Name $imageResourceGroup -Location $location
```

## Step 2 : Permissions, create user idenity and role for AIB

### Create user identity
```powerShell
# setup role def names, these need to be unique
$timeInt=$(get-date -UFormat "%s")
$imageRoleDefName="Azure Image Builder Image Def"+$timeInt
$idenityName="aibIdentity"+$timeInt

## Add AZ PS modules to support AzUserAssignedIdentity and Az AIB
'Az.ImageBuilder', 'Az.ManagedServiceIdentity' | ForEach-Object {Install-Module -Name $_ -AllowPrerelease}

# create identity
New-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $idenityName -Location $location

$idenityNameResourceId=$(Get-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $idenityName).Id
$idenityNamePrincipalId=$(Get-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $idenityName).PrincipalId

```

### Assign permissions for identity to distribute images
This command will download and update the template with the parameters specified earlier.
```powerShell
$aibRoleImageCreationUrl="https://raw.githubusercontent.com/harsha271199/imgbuilder/main/aibRoleImageCreation.json"
$aibRoleImageCreationPath = "aibRoleImageCreation.json"

# download config
Invoke-WebRequest -Uri $aibRoleImageCreationUrl -OutFile $aibRoleImageCreationPath -UseBasicParsing

((Get-Content -path $aibRoleImageCreationPath -Raw) -replace '<subscriptionID>',$subscriptionID) | Set-Content -Path $aibRoleImageCreationPath
((Get-Content -path $aibRoleImageCreationPath -Raw) -replace '<rgName>', $imageResourceGroup) | Set-Content -Path $aibRoleImageCreationPath
((Get-Content -path $aibRoleImageCreationPath -Raw) -replace 'Azure Image Builder Service Image Creation Role', $imageRoleDefName) | Set-Content -Path $aibRoleImageCreationPath

# create role definition
New-AzRoleDefinition -InputFile  ./aibRoleImageCreation.json

# grant role definition to image builder service principal
New-AzRoleAssignment -ObjectId $idenityNamePrincipalId -RoleDefinitionName $imageRoleDefName -Scope "/subscriptions/$subscriptionID/resourceGroups/$imageResourceGroup"

### NOTE: If you see this error: 'New-AzRoleDefinition: Role definition limit exceeded. No more role definitions can be created.' See this article to resolve:



```

## Step 3 : Create the Shared Image Gallery 

```powerShell
$sigGalleryName= "win2019"
$imageDefName ="win19"

# create gallery
New-AzGallery -GalleryName $sigGalleryName -ResourceGroupName $imageResourceGroup  -Location $location

# create gallery definition
New-AzGalleryImageDefinition -GalleryName $sigGalleryName -ResourceGroupName $imageResourceGroup -Location $location -Name $imageDefName -OsState generalized -HyperVGeneration "V2" -OsType Windows -Publisher 'MicrosoftWindowsServer' -Offer 'WindowsServer' -Sku '2019-datacenter-gensecond'

```

# Configure the Image Template
For this example we have a template ready to that will download and update the template with the parameters specified earlier, it will install FsLogix, OS Optimizations, Teams and run Windows Update at the end.

If you open the template you can see in the source property the image that is being used, in this example it uses a Win 10 Multi session image. 

## Windows 2019 images

```json
"publisher": "MicrosoftWindowsServer",
"offer": "WindowsServer",
"sku": "2019-datacenter-gensecond",
"version": "latest"
```

You can also change the Win2019 images available:
```powerShell
Get-AzVMImageSku -Location westus2 -PublisherName MicrosoftWindowsServer -Offer WindowsServer
```

## Download template and configure
```powerShell

$templateUrl="https://raw.githubusercontent.com/harsha271199/imgbuilder/main/armTemplate2019.json"
$templateFilePath = "armTemplate2019.json"

Invoke-WebRequest -Uri $templateUrl -OutFile $templateFilePath -UseBasicParsing

((Get-Content -path $templateFilePath -Raw) -replace '<subscriptionID>',$subscriptionID) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<rgName>',$imageResourceGroup) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<region>',$location) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<runOutputName>',$runOutputName) | Set-Content -Path $templateFilePath

((Get-Content -path $templateFilePath -Raw) -replace '<imageDefName>',$imageDefName) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<sharedImageGalName>',$sigGalleryName) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<region1>',$location) | Set-Content -Path $templateFilePath
((Get-Content -path $templateFilePath -Raw) -replace '<imgBuilderId>',$idenityNameResourceId) | Set-Content -Path $templateFilePath

```
Feel free to view the template, all the code is viewable.


# Submit the template
Your template must be submitted to the service, this will download any dependent artifacts (scripts etc), validate, check permissions, and store them in the staging Resource Group, prefixed, *IT_*.
```powerShell
New-AzResourceGroupDeployment -ResourceGroupName $imageResourceGroup -TemplateFile $templateFilePath -api-version "2020-02-14" -imageTemplateName $imageTemplateName -svclocation $location

# Optional - if you have any errors running the above, run:
$getStatus=$(Get-AzImageBuilderTemplate -ResourceGroupName $imageResourceGroup -Name $imageTemplateName)
$getStatus.ProvisioningErrorCode 
$getStatus.ProvisioningErrorMessage
```
 
# Build the image
```powerShell
Start-AzImageBuilderTemplate -ResourceGroupName $imageResourceGroup -Name $imageTemplateName -NoWait

```

>> Note, the command will not wait for the image builder service to complete the image build, you can query the status below.

```powerShell
$getStatus=$(Get-AzImageBuilderTemplate -ResourceGroupName $imageResourceGroup -Name $imageTemplateName)

# this shows all the properties
$getStatus | Format-List -Property *

# these show the status the build
$getStatus.LastRunStatusRunState 
$getStatus.LastRunStatusMessage
$getStatus.LastRunStatusRunSubState
```
## Create a VM
Now the build is finished you can build a VM from the image, use the examples from [here](https://docs.microsoft.com/en-us/powershell/module/az.compute/new-azvm?view=azps-2.5.0#examples).

Store the VM login credentials in a variable. The password must be complex.

###Azure PowerShell

```powerShell
$Cred = Get-Credential
```
Create the VM by using the image you created.

###Azure PowerShell

```powerShell
$ArtifactId = (Get-AzImageBuilderTemplateRunOutput -ImageTemplateName $imageTemplateName -ResourceGroupName $imageResourceGroup).ArtifactId

New-AzVM -ResourceGroupName $imageResourceGroup -Image $ArtifactId -Name myWinVM01 -Credential $Cred
```
# Clean Up

Delete the resource group template first, do not just delete the entire resource group, otherwise the staging resource group (*IT_*) used by AIB will not be cleaned up.

### Remove Image Template
```powerShell
Remove-AzImageBuilderTemplate -ResourceGroupName $imageResourceGroup -Name wvd10ImageTemplate
```
### Delete role assignment
```powerShell
Remove-AzRoleAssignment -ObjectId $idenityNamePrincipalId -RoleDefinitionName $imageRoleDefName -Scope "/subscriptions/$subscriptionID/resourceGroups/$imageResourceGroup"

## remove definitions
Remove-AzRoleDefinition -Name "$idenityNamePrincipalId" -Force -Scope "/subscriptions/$subscriptionID/resourceGroups/$imageResourceGroup"

## delete identity
Remove-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $idenityName -Force
```
### Delete Resource Group
```powerShell
Remove-AzResourceGroup $imageResourceGroup -Force
```
## Next Steps
