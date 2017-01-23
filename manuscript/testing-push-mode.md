# Testing Push Mode
As we described in the Part opener, in this chapter we will show you how to:

* Create a very basic configuration MOF that sets up a very basic, not-secure Pull server (securing it will require certificates, which we haven't gotten to, yet), and
* Push the configuration to a server

Whether you follow along on your own computer or not is entirely optional.

## Creating the Configuration
First, in order to set up a pull server, you need to have the xPSDesiredStateConfiguration module installed from the Powershell Gallery. Locate the module with command:

```PowerShell
Find-module xPSDesiredStateConfiguration 
```

Note that you may be prompted to download a NuGet provider.

Ensure that the module is found, and that the current version is the desired version (e.g., the latest). You could specify a specific version using the `-RequiredVersion` parameter, or list all versions using the `-AllVersions` parameter before choosing a version. This example uses the current version as of this writing (5.1.0.0). Once you find the module, install it:

```PowerShell
Install-module xPSDesiredStateConfiguration
```

The module will be placed in %systemdrive%\Program Files\WindowsPowershell\Modules, if you want to verify its existence. Once the module is installed, you can use it in the configuration. After the `Configuration` keyword, insert the following line:

```PowerShell
Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion `
  5.1.0.0 
```

Second, this simple example is setting up a non-secure - that is, HTTP instead of HTTPS - pull server. This is not a recommended configuration for any production infrastructure setup - or for any lab that tests anything that might one day go into production. This configuration is provided solely as an example for learning purposes. Remember, _you do not want your compromised pull server to be the reason your company ends up on the evening news_. If you decide to follow along with this example, do so _in an isolated, disconnected lab environment_. Seriously, OK? 

The configuration to set up a simple pull server is (and please, remember our earlier note about wacky code formatting - we've included this in the GitHub sample code repo for your convenience):

```PowerShell
Configuration PullServer {

Import-DscResource -ModuleName xPSDesiredStateConfiguration `
    -ModuleVersion 5.1.0.0

  Node Pull {
        
    WindowsFeature DSCService {
      Name = "DSC-Service"
      Ensure = 'Present'
      }

    xDSCWebService Pullserver {
      Ensure = 'Present'
      EndpointName = 'PSDSCPullServer'
      Port = '8080'  
      PhysicalPath = "$env:SystemDrive\inetpub\wwwroot\PSDSCPullServer"
      ModulePath = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Modules"
      ConfigurationPath = ` 
        "$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration" 
      State = "Started"
      DependsOn = "[WindowsFeature]DSCService"
      UseSecurityBestPractices = $false
      CertificateThumbprint = "AllowUnencryptedTraffic"
      }

    File RegistrationKey {
      Ensure = 'Present'
      DestinationPath = `
        "$env:PROGRAMFILES\WindowsPowershell\DscService\registrationKeys.txt"
      Contents = '0f9ae841-785d-4a2d-8cdf-ecae01f44cdb'
      Type = 'File'
            }
  }
}
PullServer
```

In this example, the configuration name is `PullServer`, and the node's name is `Pull` (obviously, you can't run this as-is unless that's how you've named the server upon which you plan to inflict this configuration).  It will install the DSC-Service Windows Feature, which includes IIS and other bits required for the Pull Server to work. After that, it uses the xDSCWebService resource, which is located in the xPSDesiredStateConfiguration resource that we downloaded, to configure the pull server endpoint. It will also create a file, registrationKeys.txt.  This file can contain a single GUID or multiple GUIDs.  The GUIDs are the **shared secrets** that will allow nodes to register with the pull server. 

A couple of additonal notes about this configuration:

 * The `CertificateThumbprint` setting is set to "AllowUnencryptedTraffic" so that HTTP can be used instead of HTTPS, but to use HTTPS, the thumbprint of the SSL certificate would be provided as the value instead. The certificate would need to be pre-installed in the node's Machine store by other means - this neither generates, nor installs, a certificate.
 * The `UseSecurityBestPractices` setting is set to False, because you cannot use security best practices with unencrypted traffic.
 * The `ConfigurationPath` setting is the location where the MOFs will be stored for download on the pull server. You use a local folder path that exists (or can be created) on the server.
 * The `ModulePath` setting is the location from which to pull the modules that a given configuration needs.

## Running the Configuration to Produce a MOF
Compile the configuration by loading the configuration into memory, and then executing it. The configuration block is similar to a function, where a function is loaded and then the name of the function is invoked to call the function. 

In the example, the configuration portion starts at the configuration keyword and ends at the final curly brace. The last line (`PullServer`) invokes the configuration to create the MOF. Compiling this configuration results in a `Pull.MOF` file, located in the `PullServer` subdirectory of the current working directory. Notice that the configuration name is taken for the subdirectory name, and the node name is taken for the MOF filename.

## Pushing the MOF
Assuming that the server on which you authored the configuration is also the pull server, deploy this configuration by running:

```PowerShell
Start-DscConfiguration -Path .\PullServer -Verbose -Wait
```

If your authoring machine is separate from your pull server, you can deploy the configuration from the authoring machine to the pull server remotely by adding the `-ComputerName Pull` parameter to the above command.

This will configure IIS, firewall rules, and the PSDSCPullServer endpoint. Once the configuration has completed successfully, you can verify it from the client computer - you don't even need to log on to the server! Assuming the client computer has basic network connectivity to the pull server (and again assuming you named it "pull" like we did, and that your network knows how to resolve "pull" to an IP address), launch a browser and navigate to the pull server URL:

```
http://pull:8080/PSDSCPullServer.svc
```
If the endpoint is reachable and functioning, you will receive an XML document in response.