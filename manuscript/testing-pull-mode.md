# Testing Pull Mode
This chapter picks up from the previous one. Here, we will:

* Create a basic configuration MOF and deploy it to a Pull server
* Configure a node LCM to pull the configuration
* Verify that the node pulled and applied the configuration

This section assumes that you are authoring your configurations from a machine that is neither the pull server nor the target node.

## Creating the Configuration
This example is going to set the timezone on a Windows 10 client computer named Cli1. It will be using the xTimezone module from the Powershell Gallery. We'll need to download it first.

```Powershell
find-module xTimezone | install-module
```

Once the module is installed, you can see what resources it contains - and the syntax for each - by using get-DSCResource.

```Powershell
get-DSCResource -module xTimezone -syntax
```

The output of this command shows that there is one resource in this module (xTimeZone), and it also shows you the valid properties for this resource and the syntax for using the resource in the configuration.  

```PowerShell
xTimeZone [String] #ResourceName
{
    IsSingleInstance = [string]{ Yes }
    TimeZone = [string]
    [DependsOn = [string[]]]
    [PsDscRunAsCredential = [PSCredential]]
}
```

You may need to do additional investigation in the resource's documentation to find out exactly what needs to be set. For example:

* What does "IsSingleInstance" do? 
* What string value does TimeZone accept? 

To obtain further information, check out the module itself, typically located in %systemdrive%\Program Files\WindowsPowershell\Modules. In the xTimeZone\1.6.0.0 folder, there is a readme.md file and upon reading it, you discover that not only is there information about how to find the valid strings for timezone values, there is also a sample configuration included. Not all downloaded resources are this helpful, so your mileage may vary.  

Continuing with the example, Paris seems like a great vacation destination, so we're going to configure the client computer to be in the timezone for Paris (Central European Time). Using the command provided in the readme, we find a valid string (Central European Standard Time) to use in the configuration.

```Powershell
Configuration ParisTZ {

Import-DscResource -ModuleName "xTimeZone" -ModuleVersion "1.6.0.0"  
  
  Node TimeZoneConfig {
    
    xTimeZone Paris {
      IsSingleInstance = 'Yes'
      TimeZone = 'Central European Standard Time'
      }
  }
}  

ParisTZ 
```

Notice that the node definition uses "TimeZoneConfig" as the node "name", and not the actual node name CLI1. This is because you will use configuration names on the LCM definition, and the configuration name for this config will be "TimeZoneConfig".

## Running the Configuration to Produce a MOF
Similar to running the configuration in the previous chapter, the configuration is compiled by loading the configuration into memory, and then executing it. Executing this code produces a MOF named TimeZoneConfig.mof in the ParisTZ subdirectory of the current directory.

## Deploying the MOF and Module to a Pull Server
Next, we create a checksum for the MOF file in the same folder as the MOF. This creates the TimeZoneConfig.Mof.Checksum file.

```Powershell
new-dscchecksum -path ./ParisTZ/TimeZoneConfig.mof
```

If we refer back to the Pull Server configuration for a moment, we note that we specified the Configuration Path (where the configurations go) and Module Path (where the modules go) in the pull server configuration:

```Powershell
ModulePath = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Modules"
ConfigurationPath = ` 
  "$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration" 
```

You will need these paths to know where to deploy the MOFs and modules to the pull server. You'll also need the proper rights on the pull server in order to copy the files to it. For this example, because we're using the administrative share (C$) to connect, that means you need to have administrative rights. **This is not a best practice, and is only for use in this isolated lab. Configurations provide a blueprint for your entire environment, so access to the pull server and its configurations and modules should be properly locked down using a least-privilege security model.**

Copy the MOF/checkum files to the Configuration folder.

```PowerShell
copy-item -Path .\parisTZ -Destination `
  "\\pull\C`$\Program Files\WindowsPowerShell\DscService\Configuration" `
  -recurse -Force
```

Then zip up the xTimeZone module and copy it to the pull server (all using the compress-archive command below) and create a checksum file for the zipped module. The naming convention for the zipped module on the pull server MUST be `ModuleName_VersionNumber.zip`.  

```PowerShell
Compress-Archive -Path `
  "$Env:ProgramFiles\WindowsPowershell\Modules\xTimezone\1.6.0.0\*" `
  -DestinationPath `
  "\\pull\C$\program `
  Files\WindowsPowershell\DSCService\Modules\xTimeZone_1.6.0.0.zip" `
  -force

New-DscChecksum -Path `
"\\pull\C$\program `
  Files\WindowsPowershell\DSCService\Modules\xTimeZone_1.6.0.0.zip"
```

## Creating a Meta-Configuration
In order to configure the client's LCM to get its configuration from a pull server, we'll revisit a few settings from the *Configuring the LCM* chapter.

```PowerShell
[DSCLocalConfigurationManager()]

Configuration LCM_Pull {
    
    Node CLI1 {

        Settings {
            ConfigurationMode = 'ApplyAndAutoCorrect'
            RefreshMode = 'Pull'
            }

        ConfigurationRepositoryWeb PullServer {
            ServerURL = 'http://pull:8080/PsDscPullserver.svc'
            AllowUnsecureConnection = $True
            RegistrationKey = '0f9ae841-785d-4a2d-8cdf-ecae01f44cdb'
            ConfigurationNames = @('TimeZoneConfig')
            }

        ResourceRepositoryWeb PullServerModules {
            ServerURL = 'http://pull:8080/PsDscPullserver.svc'
            AllowUnsecureConnection = $True
            RegistrationKey = '0f9ae841-785d-4a2d-8cdf-ecae01f44cdb'
            }
    }
}

LCM_Pull

```

We're using the `AllowSecureConnection = $True` setting since the pull server is only configured for HTTP. The `RegistrationKey` setting must match a registration key that exists in the RegistrationKeys.txt file on the pull server. This is the same GUID that we used when creating the pull server.  This setting is checked at registration time to ensure that the node is authorized to connect to the pull server, and later on subsequent communication with the pull server to retrieve configurations and modules.  

As with all previous examples, execute this configuration in order to create Cli1.meta.mof in the LCM_Pull subdirectory.

## Pushing the Meta-Configuration to a Node
To push the LCM configuration to Cli1:

```Powershell
Set-DscLocalConfigurationManager -ComputerName Cli1 -Path .\LCM_Pull `
  -Verbose -Force
```

This will register the LCM with the pull server!

## Pulling the Configuration from the Pull Server
The client is now ready to receive the configuration from the pull server.  We could do nothing, and at the next LCM consistency check, the LCM would pull down and apply the new configuration.  But that's not really any **fun**.  You want to see the configuration get applied, so you can issue the following command to apply the configuration right now.

```Powershell
Update-DscConfiguration -ComputerName cli1 -Verbose -Wait
```

This will apply the configuration, and if all the above instructions were completed successfully, the configuration will deploy successfuly, and your client node will be in the Central European Standard Time timezone.

## Verifying the Node's State
To check the node's status:

```Powershell
get-dscconfigurationstatus -cimsession Cli1
```

This command will show the current status of the configuration (success if it was successful), along with the `Refresh Mode` (Pull or Push) and type of configuration update (Initial, Consistency, LocalConfigurationManager).




