# Testing Push Mode
As we described in the Part opener, in this chapter we will show you how to:

* Create a very basic configuration MOF that sets up a very basic, not-secure Pull server (securing it will require certificates, which we haven't gotten to, yet)
* Pushing the configuration to a server

Whether you follow along on your own computer or not is entirely optional.

## Creating the Configuration
First, in order to set up a pull server, you need to have the xPSDesiredStateConfiguration module installed from the Powershell Gallery.  Locate the module with command

Find-module xPSDesiredStateConfiguration 
Note, you may be prompted to download a NuGet provider.

Inspect that the module is found and that the current version is the desired version.  You could specify a specific version using the -RequiredVersion parameter, or inspect all versions using the -AllVersions parameter before choosing a version.  This example is using the current version, 5.1.0.0.  Once the module is inspected, and the desired module version is found, install the module.

Install-module xPSDesiredStateConfiguration

The module will be placed in %systemdrive%\Program Files\WindowsPowershell\Modules, if you want to verify its existence.  Once the module is installed, it can be used in the configuration.  After the Configuration keyword, insert the following line to be able to use the xPSDesiredStateConfiguration module in the configuration.

Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 5.1.0.0 

Second, this simple example is setting up a non-secure - that is, HTTP instead of HTTPS - pull server.  This is not a recommended configuration for any production infrastructure setup - or for any lab that tests anything that might one day go into production.  This configuration is provided solely as an example for learning purposes.  Remember, _you do not want your compromised pull server to be the reason your company ends up on the evening news_.  

The configuration to set up a simple pull server is provided below.

    Configuration PullServer {

    Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 5.1.0.0

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
            ConfigurationPath = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration" 
            State = "Started"
            DependsOn = "[WindowsFeature]DSCService"
            UseSecurityBestPractices = $false
            CertificateThumbprint = "AllowUnencryptedTraffic"
            }
        }
    }
    PullServer

In this example, the configuration name is PullServer, and the node's name is Pull.  It will install the DSC-Service Windows Feature, which included IIS and other bits required for the Pull Server to work.  After that, it uses the xDSCWebService resource, which is located in the xPSDesiredStateConfiguration resource that was downloaded, to configure the pull server endpoint.  A couple of notes about this configuration:
 * The "CertificateThumbprint" setting is set to "AllowUnencryptedTraffic" so that HTTP can be used instead of HTTPS, but to use HTTPS, the thumbprint of the webserver certificate would be provided as the value instead.
 * The "UseSecurityBestPractices" setting is set to False, because you cannot use its security best practices settings with unencrypted traffic.
 * The "ConfigurationPath" setting is the location where the MOFs will be stored for download on the pull server.
 * The "ModulePath" setting is the location where the modules that are needed for a given configuration will be pulled from.

To deploy this configuration to the pull server, run:
Start-DscConfiguration -Path .\PullServer -Verbose -Wait

## Running the Configuration to Produce a MOF
The configuration is compiled by loading the configuration into memory, and then executing it.  This is similar to a function, where a function is loaded and then the name of the function is invoked to call the function.  In the example, the configuration portion starts at the configuration keyword and ends at the final curly brace.  The last line (PullServer) invokes the configuration to create the MOF.  Compiling this configuration results in a Pull.MOF file located in the PullServer subdirectory of the current working directory.

## Pushing the MOF
To deploy this configuration to the pull server, run:
Start-DscConfiguration -Path .\PullServer -Verbose -Wait

This will configure IIS, firewall rules, and the PSDSCPullServer endpoint.  Once the configuration has completed successfully, it can be verified from the client computer.  Assuming the client computer has basic network connectivity to the pull server, launch a browser and navigate to the pull server URL:
http://pull:8080/PSDSCPullServer.svc

If the endpoint is reachable and functioning, you will receive back an XML document.