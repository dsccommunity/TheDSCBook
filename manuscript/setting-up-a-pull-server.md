# Setting Up a Pull Server
By now, you should have almost all the basics that you need to begin using DSC. At this point in the book, I'm going to start shifting from simple examples over to more production-scale ones, and that means we're going to need a pull server. _True_, you won't always use a pull server in production. Sometimes, you'll doubtlessly have nodes to which you simply push configurations. That's fine - push mode is easy. But in what I expect will be many more cases, you'll want to use pull mode, and so that's where I'll start.

## Before You Begin
Know that Microsoft has [designated the "native" Pull Server as EOL, or "End of Life."](https://blogs.msdn.microsoft.com/powershell/2018/04/19/windows-pull-server-planning-update-april-2018/) That means they're not developing any further features, and may not support it for as long as you might want. Their "official" answer is for you to use the DSC services built into Azure Automation, or to find a third-party alternative (like the "Tug" open-source pull server). 

## Reprising the Roles
Remember that a single computer, physical or virtual, can run more than one "pull server." After all, a pull server is just an IIS website, and IIS can certainly host more than one website. Should you want to host multiples, you'll have to figure out how to differentiate them. That's a purely IIS question, though. Each website needs some unique combination of IP address, port, and host name, so you can set that up however you want. Since this book isn't about getting gnarly with IIS, I'm going to stick with a single website.

However, DSC does support a second feature, the Reporting Server, which I'll also set up. If you don't remember all the caveats about the Reporting Server, do go back and read up, so that you have a reasonable expectation for what it can do.

## A Word of Caution
We'll be using the xPSDesiredStateConfiguration module to create a pull server. Thing is, some versions of that module have been epically broken, especially when running on Server Core. Microsoft resolved the problems, but this highlights the **absolute necessity** of checking PowerShell Gallery for the latest bits, **before** you try and do anything. And, if you run into problems, hop into that module's repo on GitHub.com/powershell and see if anyone's reported a relevant bug against that version. If not... file a bug yourself.

## Step 1: Install the Module
Run this:

```
Install-Module xPSDesiredStateConfiguration
```

Or, if it's already installed, make sure it's updated (Update-Module).

## Step 2: Get an SSL Certificate
There is no legitimate reason, even in a lab environment, to run a non-secure pull server. None. "But, it's only a lab!" Yeah, but the point of a lab is to set up _what you plan to deploy in production_. So use SSL. Even if you're just "playing around," you should be playing around with it the right way. You don't drive without a seatbelt and you don't run pull servers without SSL. Seriously. Unless your _specific goal_ is to get your company on CNN when you get hacked, then you don't run a non-SSL pull server. If you _do_ run a non-SSL pull server, I kind of hope you go to jail. I really do.

A self-signed certificate will not work. You need an SSL certificate issued from a Certificate Authority that all of your nodes will trust. That means either a commercial CA, or more likely an internal CA that your computers are all already configured to trust. Don't have an internal CA? Time to stop operating like its 1999. Get with the program and set up a root CA. It's fun and easy.

Install the certificate to the local _machine_ store, not your _user_ store. That means you should be able to find the certificate by looking in CERT:/LocalMachine/My in PowerShell. Once you verify that the certificate is installed, copy its thumbprint (a GUID) to the clipboard.

## Step 3: Make a GUID
Run [guid]::newGuid() to generate a new GUID. This is going to be the registration key for the pull server. In the chapter on configuring the LCM, I cover where you'll use this GUID again, so that your nodes can "log in" to the pull server.

## Step 4: Set Up DSC
I'm starting by assuming that your pull server computer is running WMF v5 or later already. I'm going to build a configuration that turns on the pull server functionality, and I assume that SERVER1 is the name of the new pull-server-to-be.

```
configuration GoGoDscPullServer
{ 
    param  
    ( 
            [string[]]$NodeName = 'SERVER1', 

            [ValidateNotNullOrEmpty()] 
            [string] $certificateThumbPrint,

            [Parameter(Mandatory)]
            [ValidateNotNullOrEmpty()]
            [string] $RegistrationKey 
     ) 


     Import-DSCResource -ModuleName xPSDesiredStateConfiguration 

     Node $NodeName 
     { 
         WindowsFeature DSCServiceFeature 
         { 
             Ensure = 'Present'
             Name   = 'DSC-Service'             
         } 

         xDscWebService PSDSCPullServer 
         { 
             Ensure                  = 'Present' 
             EndpointName            = 'PSDSCPullServer' 
             Port                    = 8080 
             PhysicalPath            = "$env:SystemDrive\inetpub\PSDSCPullServer" 
             CertificateThumbPrint   = $certificateThumbPrint          
             ModulePath              = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Modules" 
             ConfigurationPath       = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration" 
             State                   = 'Started'
             DependsOn               = '[WindowsFeature]DSCServiceFeature'
             UseSecurityBestPractices = $True                         
         } 

        File RegistrationKeyFile
        {
            Ensure          = 'Present'
            Type            = 'File'
            DestinationPath = "$env:ProgramFiles\WindowsPowerShell\DscService\RegistrationKeys.txt"
            Contents        = $RegistrationKey
        }
    }
}

GoGoDscPullServer -cert paste_ssl_thumbprint_here -reg paste_reg_key_here
```

At the very bottom is the line of code that runs the configuration. You'll notice I'm passing in two parameters: the certificate thumbprint of the SSL certificate, and the registration key GUID you generated earlier. Let's walk through some of what's hardcoded in the configuration.

**SPECIAL NOTE:** There can be times when it's helpful to have the pull server running in HTTP, and that's when you're trying to troubleshoot it and need to do a packet capture using a tool like Wireshark. Specify a CertificateThumbprint of "AllowUnencrypted" to do that. DO NOT do that anywhere but in a walled-off lab environment, please! And, don't forget that the LCM on your nodes must be configured to allow unsecure communications, and you'll need to change the download manager URLs in the LCM to use http:// instead of https://. This is how I created some of the packet captures in the Troubleshooting chapter.

To begin with, there are several paths:

```
PhysicalPath = "$env:SystemDrive\inetpub\PSDSCPullServer" 
ModulePath = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Modules" 
ConfigurationPath = "$env:PROGRAMFILES\WindowsPowerShell\DscService\Configuration" 
```

PhysicalPath is where the pull server endpoint will be installed. ModulePath is where you'll drop your ZIPped-up DSC resource modules so that nodes can download them. ConfigurationPath is where you'll drop all your MOF files.

You can also specify the port on which the pull server will run. Notice that 8080 is nonstandard (it isn't 80 or 443); you can use whatever you like.

```
Port = 8080 
```

You also can specify the location of the Registration Keys file. You'll notice in the example that it's being pre-populated with the Registration Key you generated previously; the file can actually contain as many keys as you want.

```
DestinationPath = "$env:ProgramFiles\WindowsPowerShell\DscService\RegistrationKeys.txt"
```

There's some argument in the community about how to handle these. Some folks are comfortable creating one key and letting all their nodes use it. Others want to generate a unique key for each node. I think the latter is a lot of work and maintenance waiting to happen, and in a large environment I have some concerns about the overhead of a large key file. It's not presently clear what the potential risks could be in using one, or a small number of, registration keys.

There's another additional required setting for the pull server configuration: **UseSecurityBestPractices**. This is set to either $true or $false.  This setting, when set to $True, will disable all insecure protocols (at this time, all SSL protocols and TLS 1.0). We've seen some people have problems with the pull server when configured this way (specifically, some nodes not being able to register), requiring them also to add **DisableSecurityBestPractices='SecureTLSProtocols'** to disable TLS (which does not disable SSL). Note that these may not affect an _existing_ pull server; if you're having problems and need to reconfigure, you may need to remove an existing pull server entirely and rebuild it. Or, go into the registry key affected by the setting and remove any keys containing "TLS" in them. Essentially, UseSecurityBestPractices sets that registry key, HKLM:\\SYSTEM\\CurrentControlSet\\Control\\SecurityProviders\\SCHANNEL, to enforce stronger encryption - but it can negatively affect older clients that don't support those. 
https://support.microsoft.com/en-us/kb/245030 and 

https://technet.microsoft.com/en-us/library/dn786418(v=ws.11).aspx contain more detail.

## Step 5: Run and Deploy the Config
Running the above script (assuming you've adjusted any values you want to) will produce ./GoGoDscPullServer/SERVER1.mof. You can now push that to your prospective pull server:

```
Start-DscConfiguration -Path ./GoGoDscPullServer -ComputerName SERVER1
```

This'll take a while to run; you may want to add -Verbose and -Wait to the command to watch it all happen. If IIS isn't installed on the server, it'll get installed (as a pre-requisite of the DSC pull server feature), and that's what can take several minutes.

## Confirming the Setup
When everything's done, you should be able to view the PhysicalPath on the pull server to confirm that the DSC service endpoint (a .svc file) exists, and you can remotely use IIS Manager to attach to the server, confirm that IIS is running, and confirm that the website details (port, certificate, and so on) are correct.

And I'm deadly serious about the SSL thing. There's an upcoming chapter on DSC security where I'll dig into more specifics on why SSL is so important, and even make a case for client certificate authentication on the pull server.

## Life Choices
By default, Pull Server (in v5 and later) uses an EDB database. That's the database architecture used by Exchange Server and Active Directory, and it was the only option supported on early versions of Nano Server (when we all thought Nano Server would be a standalone install option one day). EDB databases *require maintenance*, meaning that if you don't clear the logs or set them to auto-cycle, it'll fill up the disk. Yikes. Not a wholly popular decision.

It can also be configured to use the "other" JET database format, MDB, which is what you know and love from Microsoft Access. The less said about that the better, although at least it doesn't fill up your disk.

More recently (April 2018, in Windows Server build 17090), they added the ability to instead use a SQL Server, which is a great option that we recommend heartily. You can use SQL Express, or an existing SQL Server installation on your network. When you create your xDscWebService configuration items, add two keys:

```
SqlProvider = $true 
SqlConnectionString = "Provider=SQLNCLI11;Data Source=(local)\SQLExpress;User ID=SA;Password=Password12!;Initial Catalog=master;"
```

That connection string obviously needs to be valid in your environment; this example is using a local SQL Express instance, and logging in with the horrible, you-should-never-do-this "SA" account. We strongly recommend creating a dedicated account for the pull server to log into SQL Server with or, best yet, getting Windows Integrated authentication to work. But that's a SQL Server issue, not a PowerShell one, so we'll leave it to you and your DBA.

