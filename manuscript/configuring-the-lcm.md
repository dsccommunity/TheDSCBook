# Configuring the LCM
The Local Configuration Manager, or LCM, is the "agent" for DSC. It lives and runs on each target node, accepts configurations (via either pull or push), and executes those configurations. In WMF v4, the LCM ran as a Scheduled Task; as of v5, that's not the case. You can't schedule _when_ the LCM will run, but you can schedule some parameters related to _how often_ it runs. That's what this chapter is all about, along with the LCM's other configuration options.

## Checking the Configuration
Run Get-DscLocalConfigurationManager to see the current configuration. Here's a default Windows 10 computer (in early July 2016; defaults may change in subsequent updates, so it's important not to treat this example as a reference):

```
ActionAfterReboot              : ContinueConfiguration
AllowModuleOverWrite           : False
CertificateID                  :
ConfigurationDownloadManagers  : {}
ConfigurationID                :
ConfigurationMode              : ApplyAndMonitor
ConfigurationModeFrequencyMins : 15
Credential                     :
DebugMode                      : {NONE}
DownloadManagerCustomData      :
DownloadManagerName            :
LCMCompatibleVersions          : {1.0, 2.0}
LCMState                       : Idle
LCMStateDetail                 :
LCMVersion                     : 2.0
StatusRetentionTimeInDays      : 10
PartialConfigurations          :
RebootNodeIfNeeded             : False
RefreshFrequencyMins           : 30
RefreshMode                    : PUSH
ReportManagers                 : {}
ResourceModuleManagers         : {}
PSComputerName                 :
```

Let's walk through each of those. But, before we do that, I have to point out that this set of configuration properties is from a _Windows_ computer. Microsoft produces an LCM for Linux, too, and that LCM has a smaller set of properties (at least, as of this writing; it's unclear at this point how hard Microsoft will try to keep the Linux LCM in feature-lockstep with the Windows LCM). 

Also, notice the lack of an AgentId - that's because this is a fresh machine, and it doesn't yet have one.

### ActionAfterReboot
This determines what the LCM will do after the computer is rebooted. The default, ContinueConfiguration, will let the LCM pick up where it left off and continue processing the current configuration. You can also choose StopConfiguration, which, as the name implies, stops the configuration from running further. I imagine you'd really only use StopConfiguration in certain troubleshooting or lab situations.

### AllowModuleOverWrite
When set to False - which is the default - the LCM _will not download any DSC resource modules that already exist locally_. You might wonder why the heck that's the default. Well, it's for safety. The presumption is that, once you put a resource onto a node, the node depends on that exact resource in order to run configurations properly. If you change or update a resource, you're meant to apply a new version to it. Your configurations can then reference that new version, and if properly configured, the LCM will download that new version. This AllowModuleOverwrite settings only applies to downloading new copies of the exact same module. In a test environment, where you're frequently iterating modules, you'll probably set this to True.

Why would you not set this True in production? Well... predictability and stability. If an LCM is allowed to overwrite local modules, that can be convenient for you when making ad-hoc changes. But that subverts the entire best practice of code testing, versioning, and so on. Ideally, if Configuration A relies on Module B version 1.2, you don't want Module B version 1.3 showing up out of the blue. It might not work the way Configuration A expects - for example, Module B version 1.3 might require different properties or property values, which Configuration A doesn't provide. The correct practice would be to deploy Module B v1.3 as a separate thing, living independently of, and side-by-side with, v1.2. You would then author Configuration C, which specifically references Module B v1.3, and deploy Configuration C to your nodes. That way you always know what each node is using, and troubleshooting becomes a lot easier.

### CertificateID
This is the certificate thumbprint (which you can obtain by exploring the CERT: drive) of the encryption certificate that the LCM will use to decrypt any encrypted information from its MOF. Most commonly, this is used to decrypt encrypted logon credentials. We'll cover certificates pretty thoroughly in an upcoming chapter. **This is really important**: If you specify a ConfigurationID, you can run against a v4 or v5 pull server, and the v5 pull server will follow v4 behaviors. If you omit the ConfigurationID, whether you include Configuration Names or not, the pull server must be v5 or later, will behave as v5 or later, and requires a registration key to be specified in a Repository section.

### ConfigurationDownloadManagers
This is a collection of pull servers - that is, servers from which MOF files will be pulled. 

### ConfigurationID
This is a Globally Unique Identifier (GUID) that you assign to the LCM. In WMF v4, this was the only way to tell the LCM which MOF to pull from a pull server. In WMF v5, you can also use configuration names. However, when using ConfigurationID, the node will essentially look on the pull server for a MOF file whose filename is the same GUID. Note that a GUID isn't just a random string of hexadecimal characters; it's a specific thing that must be generated. In PowerShell, you can use [guid]::newGuid() to generate one.

As of this writing, there's currently a bug in the LCM. If you plan to use Push mode, you _must_ still define a ConfigurationID, even though it'll never actually be used for anything. Otherwise, you'll get registration failure messages. You can skip this step for nodes that were _previously_ registered with a pull server using ConfigurationNames and a RegistrationKey. Even though this bug will be fixed at some point, I'm now in the habit of defining a ConfigurationID for nodes that will live in Push mode. Although, be careful. Supplying a ConfigurationID forces the node into the WMF4 pull server protocol, which means any pull server you _do_ assign will see it as a WMF4 node, not a WMF5+ node.

### ConfigurationMode
Push or Pull. As easy as that. This defaults to Push, meaning the LCM just sits and waits for you to hit it up with a configuration.

### ConfigurationModeFrequencyMins
This controls how often the LCM will re-evaluate its current configuration. It defaults to, and has a minimum value of, 15 minutes. This is important, conceptually, because it means the LCM cannot guarantee _continuous_ compliance with your desired configuration. If someone changes something, that change can "last" for 15+ minutes until the next consistency check. So in cases where the "desired configuration" could mean the difference between literal life or death, or someone going to jail or not, DSC might not be the right technology. In _many_ cases, you could extend this to 24 hours or more (calculated in minutes), for cases when you simply need a once-a-day check on configuration compliance.

### Credential
This appears to be obsolete, since the individual resource blocks (which we'll get into in a bit) can define this on a per-instance basis.

### DebugMode
Whether or not the LCM's debug mode is on or off ($True or $False).

### DownloadManagerCustomData
This appears to be obsolete in v5.

### DownloadManagerName
This appears to be obsolete in v5.

### LCMCompatibleVersions
The LCM versions that this LCM is compatible with. For example, LCM 2.0, which ships in WMF v5, is compatible with LCM 1.0, meaning it can accept the same configuration documents that the older LCM accepted, in addition to newer, 2.0-only ones.

### LCMState
This tells you what the LCM is currently up to:

* Idle
* Busy
* PendingReboot
* PendingConfiguration

### LCMStateDetail
This is usually a more descriptive version of what the LCM is doing right then, like "LCM is applying a new configuration."

### LCMVersion
The version of the LCM.

### StatusRetentionTimeInDays
This controls how long the LCM will retain current status information. The default is 10 days.

### PartialConfigurations
This doesn't presently do anything.

### RebootNodeIfNeeded
When set to True, the LCM will automatically restart the target node if a DSC resource indicates that a restart is required to finalize the configuration. This is set to False by default, on the very reasonable grounds that most people want a little more control over when their servers restart.

### RefreshFrequencyMins
This controls how often the LCM will check the pull server for new MOFs. It defaults to 30 minutes, and in WMF v4 must be set to an even multiple of the ConfigurationModeFrequencyMinutes value (15 * 2 = 30, hence the default of 30). This value should always be larger than the ConfigurationModeFrequencyMinutes.

### RefreshMode
This controls the LCM's refresh mode. The following values are allowed:

* Disabled. The LCM does not run. This is perhaps most often used in cases where a third-party management technology, like Chef, is actually running the show. Chef can use DSC resource modules under the hood, but it doesn't want the actual LCM stepping in and interfering.
* ApplyOnce. The LCM applies the current configuration, and then stops running until manually run.
* ApplyAndMonitor. The LCM applies the current configuration, and re-checks it on the ConfigurationModeFrequencyMinutes value. It does not attempt to fix the configuration, but will report a status to a Reporting Server if so configured.
* ApplyAndAutoCorrect. The LCM applies the current configuration, and re-checks it on the ConfigurationModeFrequencyMinutes value. It will attempt to fix out-of-compliance configuration settings, and if configured will report status to a Reporting Server.

### ReportManagers
This is a collection of configured reporting servers.

### ResourceModuleManagers
This is a collection of pull servers from which modules (DSC resources) can be downloaded.

## Changing the Configuration
To change the configuration of an LCM, even the local one, you need a _meta-MOF_. This is a special MOF that you can create by writing a special PowerShell configuration script. Meta-MOFs are pushed to their node, using the Set-DscLocalConfigurationManager cmdlet (which uses CIM sessions for connectivity). A meta-MOF configuration script might look something like the following:

```
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
        Settings
        {
            RefreshMode = 'Push'
        }
    }
}
```
In this case, the [DSCLocalConfigurationManager()] bit tells PowerShell that this is to be a meta-MOF, which are generated somewhat differently than a normal configuration MOF. The name of the configuration, LCMConfig, is arbitrary. You simply need to specify that name - usually at the very bottom of the script file - to run the config and generate the MOF (it's basically like running a function). Within the Settings{} block is where you can put whatever settings you like from the list above.

## Deploying the LCM Configuration
Once you've run that and generated a localhost.meta.mof, you can use Start-DscLocalConfigurationManager to "push" the meta-MOF to whatever node you want. There's no need to rename the file.

```
Set-DscLocalConfigurationManager -Path ./localhost.meta.mof -ComputerName NODE1,NODE2
```

It's also possible to "inject" the meta-MOF into a node by copying it into the place where the LCM looks for it (C:/Windows/System32/Configuration/metaconfig.mof). For example, this is a great way to help configure the LCM as part of an automated VM build, since you can use things like the Hyper-V integration cmdlets to copy files into the VM. 

**NOTE:** I should point out that there can be a bunch of caveats around "injecting" a MOF by simply copying a file. I've found it's nearly always better to push the meta-MOF to the LCM as part of an automated build process, or just copying the meta-MOF to an arbitrary local file on the node and having the VM "push" that to itself. And keep in mind that you can also "sort of push" a meta-MOF into a VM node, even if WS-MAN isn't enabled on the node, by using integration components like those provided by Hyper-V. If you want to read more on injection, though, have a look at http://dille.name/blog/2014/12/07/injecting-powershell-dsc-meta-and-node-configurations/ and https://blogs.msdn.microsoft.com/powershell/2014/02/28/want-to-automatically-configure-your-machines-using-dsc-at-initial-boot-up/, although those articles refer to WMF v4, so the actual examples are slightly outdated.

## Specifying Configuration Pull Servers
The basic settings I covered above are only the beginning of what you'll need to configure. One thing you're probably going to want to do is configure one or more pull servers, from which the LCM will pull its configuration MOF (or MOFs, if you're using partial configurations). So in addition to the Settings{} block in the meta-MOF, you can have one or more pull server sections.

```
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
		Settings {
			RefreshMode = 'Pull'
		}

		ConfigurationRepositoryWeb MainPull {
			AllowUnsecureConnection = $false
			ConfigurationNames = @('Server2Configuration')
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

		ConfigurationRepositoryShare FilePull {
			SourcePath = '\\organization\path\folder\'
		}
	}
}
```

The above example shows both a Web and a file share-based pull server configuration. You can mix and match; note that the file share-based version also allows a Credential property, where you can specify a credential to use to access the share (if you don't, then the share must allow anonymous, un-authenticated access). Frankly, file share-based pull servers are rare in my experience, and the offer less functionality. I'll assume you're using a Web-based pull server, which means the pull server is running IIS, and you've installed the DSC Pull Server feature on the server (we'll cover pull server setup in an upcoming chapter). 

Also notice that each block has a name. This is arbitrary, and meaningful to me; DSC doesn't care what you name them. I've used "MainPull" and "FilePull."

Web pull servers support these properties:

* AllowUnsecureConnection. This should always be $false, and is by default. This is a lot less about ensuring the connection is encrypted (although it also means that), and more about ensuring that the node is connecting to a known, trusted pull server. With this set to $true, it becomes very easy to use man-in-the-middle attacks to inject malicious configurations into your environment. When you wind up on CNN because you got p0wned by means of DSC, just remember that I Told You So, and stop using unsecure connections. 
* CertificateID. This is the GUID (thumbprint) of a local certificate, which the node will send to the pull server to authenticate the node to the server. This is a great way to prevent unknown nodes from even trying to contact your pull server. This is nothing fancier than good old IIS certificate-based authentication.
* ConfigurationNames. When set, this means you should remove the ConfigurationID GUID from the Settings{} block, switching the node over to requesting one or more MOFs by filename. Notice this is specified as an array. When you use this, you must also specify the RegistrationKey. You must also use a pull server that has WMF v5. Finally, note that while ConfigurationNames accepts an array, it is illegal to specify more than one, unless you are specifying partial configurations (covered in a bit).
* RegistrationKey. This is a GUID (use [guid]::newGuid() to generate it) that is configured in advance on the pull server. It acts as a kind of password for the node to "log in" and retrieve MOFs. The theory is that, when you just use a ConfigurationID (GUID), it's really hard for a bad actor to guess at your MOF filenames. But when you start using ConfigurationNames, which are likely to be meaningful words and phrases, guessing filenames becomes easier. The RegistrationKey makes it harder for a bad guy to download your MOFs and inspect them (which would teach them a lot about your environment and facilitate an attack). Note that I've seen some documentation which suggests that you can remove RegistrationKey once the node is registered; in analyzing the code for the Linux LCM, and in speaking with some of the team, that's not true. The RegistrationKey is used every time the node runs a consistency check and hits the Pull Server.
* ServerURL. The URL of the pull server, which should start with https:// unless you're okay being hacked. And yes, that means the IIS web site that the pull server is running under must have an SSL certificate that is issued by a root Certificate Authority that the node trusts. The URL must end in 'PSDSCPullServer.svc' if you're using the Microsoft-supplied pull server service.
* SignedItemType. New in WMF 5.1, this lets you tell the LCM to expect digital signatures from files delivered from the pull server or pushed to the LCM. This can be "Configuration", "Module", or both (e.g., "Configuration","Module"). See the chapter, "Security and DSC," for information on digital signing for these items.
* TrustedStorePath. New in WMF 5.1, this goes along with SignedItemType. This is optional; if you don't include it, the LCM will use its normal certificate trust procedures to verify signed items. However, if you provide this (e.g., CERT:/LocalMachine/DSCStore), then the LCM will use that path to retrieve trusted publishers. _This is an excellent idea_, because you can restrict the CAs that are trusted to only those you control. 

Notice that only SMB (file share) pull servers support a credential; Web pull servers do not. But Web pull servers support configuration names, certificate authentication, encryption, and lots more. They are Better In Every Way.

## Specifying DSC Resource Pull Servers
Even when configured in Push mode, a node can be configured to download DSC resource modules from a pull server. This relieves you of the burden of manually deploying resource modules. You do not need to set up a unique pull server for this purpose; whatever pull server you've already got will do fine.

```
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
		Settings {
			RefreshMode = 'Pull'
		}

		ConfigurationRepositoryWeb MainPull {
			AllowUnsecureConnection = $false
			ConfigurationNames = @('Server2Configuration')
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

		ResourceRepositoryWeb ModuleSource {
			AllowUnsecureConnection	= $false
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'			
		}
		
	}
}
```

Notice that the RegistrationKey and ServerURL are the same? I'm using the exact same pull server. I can also specify a CertificateID setting, if the pull server requires client certificate authentication. You can also define a ResourceRepositoryShare{} block, which would be for a file share-based pull server, and it accepts Credential and SourcePath properties.

Again: it's totally legal in WMF v5 to specify a ResourceRepositoryWeb{} block, completely omit any other configurations, and leave the LCM in Push mode. When you Push a configuration to the node, it'll scan to see what resources are needed. If it's missing any of those resources, the node will connect to the defined server and attempt to download the ZIPped resource module. It'll then unzip it and proceed with the configuration.

## Specifying Reporting Servers 
A reporting server provides a check-in point for the node. This is always Web-based, meaning it runs under IIS. It can be the exact same server as a Web-based pull server, if you want.

```
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
		Settings {
			RefreshMode = 'Pull'
		}

		ConfigurationRepositoryWeb MainPull {
			AllowUnsecureConnection = $false
			ConfigurationNames = @('Server2Configuration')
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

		ResourceRepositoryWeb ModuleSource {
			AllowUnsecureConnection	= $false
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'			
		}
		
		ReportServerWeb CheckInPoint {
			AllowUnsecureConnection = $false
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

	}
}
```

You can also specify a CertificateID, if the server requires client certificate authentication. Notice that the service endpoint - PSDSCPullServer.svc - is the same as it was for the regular pull server function. That's a change in WMF v5; previously, the "compliance server," as it was then known, had its own endpoint.

## Partial Configurations
The last thing you might want to do with the LCM is configure partial configurations. I've outlined previously what these are (and why I'm not sure they're a good idea), and here's how you set them up:

```
[DSCLocalConfigurationManager()]
configuration LCMConfig
{
    Node localhost
    {
		Settings {
			RefreshMode = 'Pull'
		}

		ConfigurationRepositoryWeb MainPull {
			AllowUnsecureConnection = $false
			ConfigurationNames = @('Partial1','Partial2')
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

		ResourceRepositoryWeb ModuleSource {
			AllowUnsecureConnection	= $false
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'			
		}
		
		ReportServerWeb CheckInPoint {
			AllowUnsecureConnection = $false
			RegistrationKey = '140a952b-b9d6-406b-b416-e0f759c9c0e4'
			ServerURL = 'https://mypullserver.company.pri/PSDSCPullServer.svc'
		}

		PartialConfiguration PartialOne {
			ConfigurationSource = @('[ConfigurationRepositoryWeb]MainPull')
			Description = 'Partial1'
			RefreshMode = 'Pull'
			ResourceModuleSource = @('[ResourceRepositoryWeb]ModuleSource')
		}
		
		PartialConfiguration PartialTwo {
			ConfigurationSource = @('[ConfigurationRepositoryWeb]MainPull')
			DependsOn = [PartialConfiguration]PartialOne'
			Description = 'Partial2'
			ExclusiveResources = @('OrderEntryAdmin','OrderEntryConfig')
			RefreshMode 'Pull'
			ResourceModuleSource = @('[ResourceRepositoryWeb]ModuleSource')
		}

	}
}
```

 Here's what to pay attention to:
 
 * I've defined two partial configurations, which I've arbitrarily named Partial1 and Partial2. If you go back and look at the pull server configuration, you'll see that these are now listed as the ConfigurationNames for the node. Partial1.mof and Partial2.mof must live on that pull server.
 * Each partial configuration defines which pull server it'll come from. You have to refer to a pull server that you already defined - in this case, I've only defined MainPull, and so both partials must come from there. Again, Partial1.mof and Partial2.mof must live on that pull server.
 * PartialTwo won't execute until Partial1 has finished - the DependsOn sees to that.
 * PartialTwo is the only configuration that can contain settings using the OrderEntryAdmin and OrderEntryConfig DSC resources. No other configuration  may use those resources, due to the ExclusiveResources setting.
 * Both partials specify my ModuleSource definition as the place to find any missing DSC resource modules.
 * Both partials specify that the configuration be pulled. However, it's legal for some partials to expect to be pushed. Any partials set to Push mode will simply sit and wait for you to push the MOF manually. They wouldn't have a ConfigurationSource, in that case.

As you can see, these can become _insanely_ complicated and hard to keep track of. In addition, as I've pointed out before, the actual partial configuration MOFs must be carefully - manually! - evaluated to ensure they don't contain conflicting keys or other problems. The LCM will download all the MOFs, combine them, and then scan them for problems. Any problems will result in the entire configuration - all partials! - being abandoned. 

Note that you can also configure a partial configuration to be pushed, making it possible to push some, and pull some, of the total configuration. When a partial is configured for push, the node will always use the most recently pushed one when it combines all of its partials into the total configuration.

```
[DSCLocalConfigurationManager()]
configuration PartialConfigDemo
{
    Node localhost
    {

        PartialConfiguration PushPart {
            Description = 'Configuration being pushed'
            RefreshMode = 'Push'
        }
    }
}
```

That's how the LCM would be configured to understand that one of its partial configurations will be pushed to it. The configuration you push - e.g., the filename - must match. So, in this case, I'd have to push pushpart.mof. I'd use **Publish-DscConfiguration** to push the file to the node, and then run **Update-DscConfiguration** to let any additional partials (obviously not shown in the above snippet) be pulled, and for the consistency check to begin.

And actually... that's an oversimplification, the file naming bit. Assuming the node is set to use ConfigurationNames, the filename above would be pushpart.mof. But, if you'd set the LCM to use a ConfigurationID (that is, a GUID), you'd name it pushpart.GUID.mof, inserting the GUID in the filename. 

You can also sequence your partial configurations. Check this out:

```
[DSCLocalConfigurationManager()]
configuration PartialConfigDemo
{
    Node localhost
    {
    
    	ConfigurationRepositoryWeb MyPull {
    		ServerURL = 'https://whatever:8080/PSDSCPullServer.svc'
    	}

        PartialConfiguration PushPart1 {
            Description = 'Configuration being pushed'
            RefreshMode = 'Push'
        }

        PartialConfiguration PushPart2 {
            Description = 'Configuration being pulled'
            RefreshMode = 'Pull'
            ConfigurationSource = '[ConfigurationRepositoryWeb]MyPull'
            DependsOn = '[PartialConfiguration]PushPart1'
        }
    }
}
```

I'm still omitting the Setting{} block in this snippet, but you can see that PushPart2 won't be applied until after PushPart1.  

## Versions and Troubleshooting
The syntax I've used in this chapter is all WMF v5-specific. That said, v5 LCMs can be configured using the older v4 syntax, as well. Keep in mind, however, than a v5 LCM cannot be configured to use ConfigurationNames (and registration keys) if it's talking to a v4 pull server.

If you run into problems configuring the LCM, **simplify**. Start by seeing if you can just get it running in Pull mode, using a normal configuration ID rather than a configuration name. Doing so eliminates the whole registration key process, for example, thus eliminating a possible point of trouble.

