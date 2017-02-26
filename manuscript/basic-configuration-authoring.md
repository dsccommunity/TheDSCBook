# Basic Configuration Authoring
In this chapter, we'll start digging into configuration authoring. There'll be a more advanced chapter on this later in the book, so we'll stick with the basics for now, and then build on those as we go.

If you're looking to try some of this stuff on your own, it's not hard to set up a lab environment. I'm using a Windows 10 virtual machine, running the latest version of Windows PowerShell (and I can't stress enough how important it is to make sure you've updated to the _latest and greatest_). I'm also running a Windows Server 2012 R2 server, also with the latest version of PowerShell. At the time of this chapter's writing, that's PowerShell v5.1 (which you can check by running $PSVersionTable in a console window). Neither machine belongs to a domain, and for this chapter I'm not even going to be using the server. I'll author, compile, and run my configurations entirely locally on the Win10 computer.

## Getting Started: The Configuration Block
This is the simplest part of your configuration script:

```
configuration MyConfig {
}
```

While it's technically possible to include multiple configuration blocks in a single .ps1 file, I avoid doing so. For one, Azure DSC doesn't like it. For another, I like to have my script filenames match the configuration names - so the above would be in MyConfig.ps1. 

## Adding Nodes
The next part of the configuration identifies the node or nodes that you plan to create MOFs for. At the simplest level, it can look like this:

```
configuration MyConfig {
	node SERVER2 {
	}
}
```

When I run MyConfig, I'll get SERVER2.mof as the output. Of course, this isn't the most efficient way to create configurations, because you'll often have multiple computers that need the same configuration settings. 

## Adding a Parameter Block
One way you can start to make your configurations more dynamic is to add input parameters:

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {
	}
}
```

Now, I could run something like **MyConfig -ComputerName SERVER1,SERVER2,SERVER3** and get three MOFs.

## Adding Settings
Within the Node section, you identify the configuration settings that you want to go into the MOF. Let's start with something really simple:

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {
	
		WindowsFeature Backups {
			Ensure = Present
			Name = Windows-Backup
		}
		
	}
}
```

Let's look at what that's doing:

* I've specified a configuration setting that I've named _Backups_. This is an arbitrary name, meaning it's only going to have meaning to me. I could have named it Fred, or Rock, or whatever.
* The setting uses the _WindowsFeature_ DSC resource. This happens to be a native resource on Windows Server computers that have WMF 4 or later installed, so I won't need to worry about deploying the resource.
* Within the setting, I've specified two _properties_. The properties of a setting will vary based on the DSC resource it uses. This particular DSC resource offers Ensure and Name as properties; other resources will be different. 
* I've specified values - Present and Windows-Backup - for my properties. Again, the valid values will vary depending on the DSC resource that you're using. I know that, for the WindowsFeature resource, Ensure will either be _Absent_ or _Present_, and I know that Name wants the feature name as shown by running Get-WindowsFeature. The chapter on "Finding and Using Resources" will explain how I found that out.

At the most basic level, this is all a configuration is: a bunch of settings, in a Node block, in a Configuration block.

## Adding Basic Logic
Configurations can use PowerShell's entire scripting language to make decisions. For example:

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {
	
		$features = @('windows-backup','dns-server','dhcp-server')
		ForEach ($feature in $features) {
			WindowsFeature $feature {
				Ensure = Present
				Name = $feature
			}
		}
	}
}
```

When this is run, it'll produce a MOF with three Windows features. Notice that I've used $feature for the name of the _setting_ as well as using it to identify the _feature_ name. That's because _the name of every setting must be unique within the configuration_. This probably isn't an incredibly practical example, but it does show you how you can use logic in a configuration.

**But here's the important bit:** This logic only executes when the configuration is run on your authoring machine. The final MOF won't contain the ForEach loop; the MOF will contain three static settings. MOFs aren't scripts - they don't contain logic. MOFs don't ask questions and make decisions, for the most part; they just do what they contain.

## Adding Node-Side Logic
There's one exception to the "MOFs don't contain logic" rule, and it's the special, built-in Script resource. 

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {

		Script MyScript {
			GetScript  {}
			TestScript {}
			SetScript  {}
		}
	
	}
}
```

Within the TestScript script block, you can put whatever you want, provided it returns $True or $False as its only output. If it returns $False, then the target node's LCM will run whatever's in the SetScript script block. The GetScript script block should return a hash table that describes the current configuration. GetScript can either return an empty hashtable, or a hashtable with a single key-value pair.  The key must be called "Result", and the value must be a string that represents the state of the configuration.  

The **fun thing here** is that these script blocks don't run when you run the configuration and create MOFs. Instead, the PowerShell code in those three script blocks goes right into the MOF as-is. When the MOF is applied to a target node, it's the _LCM on the target node_ that will run these. So this is a sort of way to make "dynamic" MOFs. Here's a quick example:

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {

		Script MyScript {
			GetScript  { write @{} }
			TestScript { Test-Path 'c:\Program Files\Example Folder' }
			SetScript  { Mkdir 'c:\Program Files\Example Folder' }
		}
	
	}
}
```

My GetScript returns an empty hash table; that's not uncommon. My TestScript will return $True if a folder exists, and $False if it doesn't (because that's what Test-Path itself returns). If the folder doesn't exist, my SetScript block creates the folder. Again, a very simple example, but shows you the idea.

However...

You need to be a little careful with Script resources. They can be harder to maintain and, if they're complex, really difficult to troubleshoot. For all but the simplest purposes, I usually prefer creating my own custom DSC resource. Honestly, doing so requires like four extra lines of code on top of just writing a Script resource, and you end up with something that's easier to maintain and debug. We'll obviously be covering that in an upcoming chapter.

## Documenting Dependencies
I've touched on this before, but you can help the LCM understand the order in which settings must be applied by using the DependsOn property. This is a common property, available for all DSC resources. 

```
configuration MyConfig {
	param(
		[string[]]$ComputerName
	)
	
	node $ComputerName {
	
		WindowsFeature Backups {
			Ensure = Present
			Name = Windows-Backup
		}
		
		WindowsFeature DHCP {
			Ensure = Present
			Name = DHCP-Server
		}		
		
		CustomDHCPLoader LoadDHCP {
			SourceLocation = \\companyroot\admin\dhcp\sources.xml
			DependsOn = [WindowsFeature]DHCP,[WindowsFeature]Backups
		}
		
	}
}
```

In this example, the CustomDHCPLoader setting (which is something I made up; it's not native to Windows) won't run until after the two WindowsFeature settings are confirmed to be correct. 

## Running the Configuration
I've sort of already covered this, but in most cases you'll actually run the configuration right at the bottom of the script file that the configuration lives in:

```
MyConfig -ComputerName ONE,TWO,WHATEVER
```

But in addition to any parameters _you_ define, you also get a handful of free parameters that you can use:

* **-InstanceName** isn't something you'll probably ever use. It lets you override the ID that DSC uses to manage composite configurations. and it's honestly best to not mess with it.
* **-OutputPath** accepts a folder path in which your MOF files will be created. By default, PowerShell will create a folder named after your configuration (/MyConfig, in the example I've used), and put the MOFs in there.
* **-ConfigurationData** accepts a hash table of configuration data, which you can use to drive logic decisions within the configuration. This is a big topic, and there's a whole chapter on it later.

## Deploying the MOF
Once you've got the MOF authored, you're ready to deploy it. You might do that in Push mode by using `Start-DscConfiguration`, or in Pull mode, which we cover in the "Deploying MOFs to Pull Servers" chapter.

## Wrapping Up
This should give you a basic idea of what configurations look like. You can do a _lot_ more with them, and we'll have a whole chapter on more advanced options later in the book. For now, I just wanted to set the stage for this portion of DSC, so that we can cover some of the other major players.
