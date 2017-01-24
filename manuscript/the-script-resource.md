# The Script Resource
One of the native resources in DSC is the Script resource. This is intended as a sort of catch-all, quick-and-dirty way of writing your own custom resource, without all the bother of building out a class- or function-based resource. It's worth noting that Script resources can be a great way of prototyping an idea, but they're terribly hard to maintain over the long haul. The strong preference is to use class- or function-based resources rather than Script resources. But... there are a couple of neat things you can do with them.

## The Basics
Take a look at this example (for your convenience, this is available in http://github.com/concentrateddon/TheDSCBookCode):

```
Configuration TestUsing {

param (
 [Parameter()] 
 [ValidateNotNull()] 
 [PSCredential]$Credential = (Get-Credential -Credential Administrator)
)

 node $AllNodes.Where({$_.Role -eq 'DC'}).NodeName {
    
  $DomainCredential = New-Object `
   -TypeName System.Management.Automation.PSCredential `
   -ArgumentList ("$($node.DomainName)\$($Credential.UserName)", `
    $Credential.Password)

  script CreateNewGpo {
   Credential = $DomainCredential
   TestScript = {
    if ((get-gpo -name "Test GPO" `
                 -domain $Using:Node.DomainName `
                 -ErrorAction SilentlyContinue) -eq $Null)
    {
     return $False
    } else {
     return $True
    }
   } #Test
   
   SetScript = {
    new-gpo -name "Test GPO" -Domain $Using:Node.DomainName
   } #Set
   
   GetScript = {
    $GPO= (get-gpo -name "Test GPO" -Domain $Using:Node.DomainName)
    return @{Result = $($GPO.DisplayName)}
    
   } #Get   
 } #Script
} #Node
} #Configuration

TestUsing -OutputPath "C:\Using Example" -ConfigurationData ".\mj.psd1"
```

That'll create a simple MOF that uses a Script resource. Note that the Script setting has three properties: a GetScript, a TestScript, and a SetScript. These don't accept any input parameters, and they're run by the LCM just like the Get/Test/Set portions of a custom resource. The TestScript is expected to return $True or $False; the GetScript is expected to return a hash table with the current configuration, and the SetScript is expected to set the desired configuration.

## Cool Tricks
Notice the $Using trick in the example shown above? To make sense of that, let's first look at the ConfigurationData we fed to the configuration script. It's included in a separate file, and looks like this:

```
@{
    AllNodes = @(
        @{
            NodeName = '*'
            
            # Common networking
            InterfaceAlias = 'Ethernet'
            DefaultGateway = '192.168.2.1'
            SubnetMask = 24
            AddressFamily = 'IPv4'
            DnsServerAddress = '192.168.2.11'
                       
            # Domain and Domain Controller information
            DomainName = "Company.Pri"
            DomainDN = "DC=Company,DC=Pri"
            DCDatabasePath = "C:\NTDS"
            DCLogPath = "C:\NTDS"
            SysvolPath = "C:\Sysvol"
            PSDscAllowPlainTextPassword = $true
            PSDscAllowDomainUser = $true 
                        
            # DHCP Server Data
            DHCPName = 'LabNet'
            DHCPIPStartRange = '192.168.2.200'
            DHCPIPEndRange = '192.168.2.250'
            DHCPSubnetMask = '255.255.255.0'
            DHCPState = 'Active'
            DHCPAddressFamily = 'IPv4'
            DHCPLeaseDuration = '00:08:00'
            DHCPScopeID = '192.168.2.0'
            DHCPDnsServerIPAddress = '192.168.2.11'
            DHCPRouter = '192.168.2.1'
 
           # ADCS Certificate Services information
            CACN = 'Company.Pri'
            CADNSuffix = "C=US,L=Phoenix,S=Arizona,O=Company"
            CADatabasePath = "C:\windows\system32\CertLog"
            CALogPath = "C:\CA_Logs"
            ADCSCAType = 'EnterpriseRootCA'
            ADCSCryptoProviderName = 'RSA#Microsoft Software Key Storage Provider'
            ADCSHashAlgorithmName = 'SHA256'
            ADCSKeyLength = 2048
            ADCSValidityPeriod = 'Years'
            ADCSValidityPeriodUnits = 2
            # Lability default node settings
            Lability_SwitchName = 'LabNet'
            Lability_ProcessorCount = 1
            Lability_StartupMemory = 1GB
            Lability_Media = '2016TP5_x64_Standard_EN' # Can be Core,Win10,2012R2,nano
                                                       # 2016TP5_x64_Standard_Core_EN
                                                       # WIN10_x64_Enterprise_EN_Eval
        }
        @{
            NodeName = 'DC'
            IPAddress = '192.168.2.11'
            Role = @('DC', 'DHCP', 'ADCS')
            Lability_BootOrder = 10
            Lability_BootDelay = 60 # Number of seconds to delay before others
            Lability_timeZone = 'US Mountain Standard Time' #[System.TimeZoneInfo]::GetSystemTimeZones()
        }
<#
        @{
            NodeName = 'S1'
            IPAddress = '192.168.2.50'
            Role = @('DomainJoin', 'Web')
            Lability_BootOrder = 20
            Lability_timeZone = 'US Mountain Standard Time' #[System.TimeZoneInfo]::GetSystemTimeZones()
        }
        @{
            NodeName = 'Client'
            IPAddress = '192.168.2.100'
            Role = 'DomainJoin'
            Lability_ProcessorCount = 2
            Lability_StartupMemory = 2GB
            Lability_Media = 'WIN10_x64_Enterprise_EN_Eval'
            Lability_BootOrder = 20
            Lability_timeZone = 'US Mountain Standard Time' #[System.TimeZoneInfo]::GetSystemTimeZones()
        }
#>
        
    );
    NonNodeData = @{
        Lability = @{
            # EnvironmentPrefix = 'PS-GUI-' # this will prefix the VM names                                    
            Media = @(); # Custom media additions that are different than the supplied defaults (media.json)
            Network = @( # Virtual switch in Hyper-V
                @{ Name = 'LabNet'; Type = 'Internal'; NetAdapterName = 'Ethernet'; AllowManagementOS = $true;}
            );
            DSCResource = @(
                ## Download published version from the PowerShell Gallery or Github
                @{ Name = 'xActiveDirectory'; RequiredVersion="2.13.0.0"; Provider = 'PSGallery'; },
                @{ Name = 'xComputerManagement'; RequiredVersion = '1.8.0.0'; Provider = 'PSGallery'; }
                @{ Name = 'xNetworking'; RequiredVersion = '2.12.0.0'; Provider = 'PSGallery'; }
                @{ Name = 'xDhcpServer'; RequiredVersion = '1.5.0.0'; Provider = 'PSGallery';  }
                @{ Name = 'xADCSDeployment'; RequiredVersion = '1.0.0.0'; Provider = 'PSGallery'; }
            );
        };
    };
};
```

Honestly, the important bit there is the DomainName property under the NodeName = '\*' section. Because DomainName applies to all Nodes (e.g. the asterisk as a wildcard), each node that the configuration script processes will have a DomainName property applied.

Now go back and look at the $Using tricks in the configuration script. See how they're referring to **$Using:Node.DomainName**? We're telling the Script resource to grab the configuration data (represented by **Node**) and then get its DomainName property. In theory, this should return "Company.Pri" each time, because that's what our ConfigurationData block has for DomainName.

But keep something in mind: When the configuration script run, we're passing it the ConfigurationData block, right? So the computer running the configuration knows what the ConfigurationData is. The MOF it produces will contain the literal contents of the TestScript, SetScript, and GetScript blocks. _The code within those blocks is not executed or parsed when the MOF is produced!_ Those three blocks aren't executed until the MOF is sent to a node, and that node's LCM loads up the MOF and begins a consistency check. _So how the heck does the node's LCM know what the configuration data is?_ The answer is to look at the MOF generated by our configuration. This won't be pretty, and you're actually invited to run the code yourself to get the resulting MOF. It's actually almost impossible to reproduce the MOF in this book, in fact, because it's so wonky-looking.

What you'll find in the MOF, if you look at it in Notepad or something (keep in mind it's in the ZIP file, available as a book "extra" in LeanPub), is that each script block - GetScript, TestScript, and SetScript - has a serialized (e.g., translated to XML) version of the node's ConfigurationData. Keep in mind that GetScript, TestScript, and SetScript are all run independently, in separate PowerShell processes. That means the three of them don't "share" any data between them, which is why the ConfigurationData gets serialized into each block - meaning, the MOF contains three copies of the ConfigurationData.  So when GetScript is run, for example, the LCM deserializes the ConfigurationData back into an object, and makes it accessible via **$using.Node**. In fact, right in the MOF, you can see this happen:

```
ResourceID = "[Script]CreateNewGpo";
 GetScript = "$Node = [System.Management.Automation.PSSerializer]::Deserialize(
```

Basically, when the MOF is created, PowerShell tacks on some extra code to the start of each script block, recreating the $Node variable. **$Using.Node** becomes a way to refer to $Node, which turns out to contain the ConfigurationData that was used to build the MOF. This is a cool way to pass information into the MOF, and then _use_ that information during a consistency check on the node itself.