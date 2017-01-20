# Designing DSC
Deciding _how_ to "do" DSC in your environment is a huge deal. Decide wrong, and you could end up not getting the flexibility you need, or you could end up making things way more complicated than you need. 

There are two important things to remember when it comes to designing your DSC approach:

* There is no one approach that is right for everyone - you need to choose an approach that meets _your_ needs.
* You need to thoroughly understand the options before deciding

This chapter will help you understand what DSC is doing, how it does it, and what your options are for designing your DSC approach.

## The Players
The main actor in the world of DSC is the Local Configuration Manager, or LCM. The LCM is a little piece of software that lives on each computer ("target node") that you manage with DSC. The LCM's job is to read your configuration documents, see if the local computer is in compliance with its configuration document, and potentially fix the computer so that it is compliance. The LCM is really the main "moving part" in DSC. 

A secondary player in the DSC game is your authoring computer. This is simply the machine where you sit, and whose keyboard you press with your fingers. 

A third and optional player is a DSC Pull Server. This is a normal Windows Server (usually Windows Server 2012 or later) running Internet Information Service (IIS) and a special piece of additional software that is the Pull Server itself. Technically, you can also turn a file server into a Pull Server, but that lacks some of the functionality and security options of an IIS-based Pull Server. In this book, I'll assume you're using the IIS-based Pull Server, rather than a file server. The Pull Server is where you put all your configuration documents, as well as supporting code. You then configure target nodes' LCMs with the location of the Pull Server, and they go to the Pull Server to find their configuration documents. It's a little bit like how client computers "pull" their Group Policy objects (GPOs) from a domain controller.

The Pull Server can also host a component called the Reporting Server. This is a separate web service, also running under IIS, where target nodes' LCMs can check in with a status report. This lets nodes indicate that they're in compliance with their configuration, or that they're awaiting a restart, or that a problem has occurred. It's important, though, to understand the scope of the Reporting Server. First, that LCM status is just a single-digit code. It isn't a whole problem report, or configuration description, or anything like that. Also, there's no actual... um... "reporting." The Reporting Server tracks node status in a simple local database, and it doesn't provide any native reporting, dashboards, or anything like that. This is the "Technology vs. Tool" thing I pointed out in the Introduction - we have the technology, but no native tools for it.

So those are the players in the game. Now, you need to know the pieces that they play with.

## The Pieces
The first and main "piece" in DSC is the Managed Object Format (MOF) file. This is a plain-text file that contains your configuration specifications. It's the "configuration document" that I've loosely referred to. MOF is actually an industry-standard format developed by the Distributed Management Task Force (DMTF), an industry group that Microsoft and other vendors belong to. A MOF is not a script - it's simply a bunch of text instructions, written in a specific format that a computer can easily interpret. MOF files aren't terribly human-readable, but you can work through one if you have to. Think of them as a kind of fancy INI file. The important thing to understand is that a MOF is a static file - it doesn't (except for one hair-splitting exception) contain any executable code, any logic, or any decision-making capability. In that way, a MOF is like a Word document - it's just a static description of what a computer should look like.

So how do you create a MOF file? Well, you could use Notepad, if you're the nitty-gritty, I-eat-nails-for-breakfast kind of person. But nobody does that. Another, easier way to create a MOF is a _PowerShell configuration script_. A configuration script is a special kind of script that, when you run it, produces one or more MOF files. Configuration scripts can have incredibly complex logic that governs the final MOF contents - or a configuration script can be essentially static, contain no logic, and produce the same MOF file every single time you run it. 

There's a special flavor of MOF called a _meta-configuration MOF_. Again, these can be created in Notepad or, more likely, by writing and running a PowerShell configuration script. These meta-MOFs are used to change the configuration of the LCM itself. They're how you tell the LCM how often to run, where its Pull Server lives, and so on.

The second "piece" in DSC is the collection of _DSC resources_ used by the LCM. A resource is usually just a PowerShell script, although they can also be written in a .NET language like C# and compiled into a DLL. One or more resources can be collected into a _DSC resource module_, which is what you actually distribute. Your authoring computer will need a copy of any DSC resource modules that you plan to use, and each target node will need a copy of any DSC resource modules that are used by its configuration.

DSC resources are how the LCM checks the node's current configuration, and how it fixes the configuration if needed. Each DSC resource consists of three basic components: Get, Set, and Test. So let's say you have a resource designed to handle the installation and removal of Windows features (this exists - it's the WindowsFeature resource). Your MOF might instruct the LCM to ensure that the Web-Server Windows feature is present. When the LCM reads that MOF, it will load the WindowsFeature resource, and run its Test function, passing the Web-Server feature name. The Test function will see if Web-Server is installed or not, and return either True or False. If Test returns False, then the LCM knows that the node is not in compliance with the configuration MOF. If it's been configured to fix that, then the LCM will call the Set function of WindowsFeature, again passing the Web-Server feature name. The Set function will install the feature, bringing the node into compliance.

To quickly summarize that: the MOF contains a description of what you want the node to look like, and the DSC resources know how to check the configuration and, if necessary, fix it. So your abilities with DSC are limited only by the DSC resources that you have. Anything you have a resource for, you can manage via DSC. And, because DSC resources are just PowerShell scripts, you can always create a DSC resource for whatever you might need.

## The CIM Connection
It's interesting to know that, under the hood, DSC is based almost entirely in CIM. For example, when you run a command to send a configuration MOF to a node, that connectivity happens over a CIM session, just like the ones used by commands like Get-CimInstance.  

## Uniqueness in MOFs
There's an important technical detail about MOFs that you need to know before we go any further. Take a look at this chunk of a PowerShell configuration script:

```
WindowsFeature DnsServer {
	Name = 'DNS'
	Ensure = 'Present'
}
```

This is what I call a **configuration setting**. It specifies the configuration for one specific item. The **WindowsFeature** part of that is the name of a resource, which would be contained in a resource module, and which would be locally installed on the node. Following that is the **name** for this configuration setting. I made this up - "DnsServer" is meaningful to me, so that's what I used. This name does need to be unique - in the final MOF, no other configuration setting can use the same name. But I could also have used "Fred" or "Whatever" as legal names - DSC doesn't care what the name is, so long as it's unique.

Inside the configuration setting are its **properties**, including Name and Ensure. These properties and their values will be passed to the DSC resource by the LCM, and tell the resource what, specifically, it's supposed to be checking and doing. Every DSC resource is designed to have one or more _keys_, which are the properties they use to identify some manageable component of the computer. In the case of the WindowsFeature resource, the key happens to be the **Name** property. Here, I've specified "DNS" for the name, because "DNS" is the name of the Windows Feature I want installed.

Because **Name** was designed to be the key for the WindowsFeature resource, _it must be unique for all instances of the WindowsFeature resource within the final MOF_. This restriction is designed to stop you from doing stuff like this:

```
WindowsFeature DnsServer1 {
	Name = 'DNS'
	Ensure = 'Present'
}
WindowsFeature DnsServer2 {
	Name = 'DNS'
	Ensure = 'Absent'
}
```
This doesn't make any sense! You want DNS installed - but not installed? Make up your mind! Because this kind of configuration can throw the LCM into an infinite loop of installing and uninstalling, DSC won't allow it. In fact, during its pre-scan of the MOF, the LCM will spot this duplicate "DNS" key and abort. It'll actually refuse to run anything in the MOF, just because of this one problem.

This is really, really important - and it can be tricky, because as you physically write a configuration script, there's no easy or obvious way to see exactly what the key properties are. You _can_ look it up, and I'll show you how, and you have to be very careful not to create duplicates.

## Getting the MOF to the LCM
There are three ways of getting a MOF to an LCM:

* Manual file copy. You simply have to drop the MOF in the correct location (and run a command to encrypt it, but we'll get into that later). The LCM will see it, and will process it on its next run.
* Push mode. Usually from your authoring computer, you'll run Start-DscConfiguration to _push_ one or more MOF files to target nodes, with a maximum of one MOF per node. This is a one-time operation that relies on Remoting to provide the connection to the nodes.
* Pull mode. If you configure the LCM to do so, it can look to one (or more, which I'll explain in a bit) Pull Servers for configuration MOFs. By default, the LCM will check for a new MOF every 30 minutes. If it finds one, it'll download it and start processing it.

Pull Servers can also contain ZIPped copies of DSC resource modules. When a node gets a MOF, it does a quick pre-scan of the MOF to make sure any needed DSC resources are already locally installed. If they're not, then the LCM can be configured to download needed DSC resource modules from a Pull Server. The LCM then unzips and installs those locally, and then runs its MOF. The LCM can be configured to "pull" resource modules in this way, even if it's not "pulling" MOFs. In other words, you can configure the LCM to wait for you to _push_ it a MOF, but still download needed DSC resource modules from a Pull Server.

## Configuration Variations
So far, I've described the DSC process with the assumption that you're creating one MOF for each target node. That's a good way to understand how DSC works, but it obviously might not be ideal for every environment. After all, you probably have many different kinds of servers, but a lot of them probably have many similarities. So some of the first design questions people often ask about DSC tend to revolve around modularizing the MOFs.

By the way, in many of the following examples, I'm going to be referring to DSC resources that are fictional, and even using real ones with values that might not work in the real world. The goal of this chapter is to help you understand the design decisions you need to make, not to provide a reference of resources or syntax. So I've tried to construct these examples in a way that best illustrates the _concepts_. I just wanted you to know that, so you don't try copying these into a real configuration. I'll provide plenty of _functional_ examples in later chapters.

### One MOF, Many Nodes
One thing you can do is simply assign the same MOF to multiple nodes. Provided the MOF doesn't try to configure any must-be-unique information, like computer name or IP address, this is totally legal. You can either push the same MOF to more than one node, or you can configure multiple nodes to pull the same MOF from a Pull Server. This works well in situations where you _want_ multiple nodes configured identically, such as a bunch of web servers that are all part of the same load-balanced set.

### Configuration Data Blocks
When you run a configuration script, you can pass in a _configuration data_ block. This is a data structure that lists the nodes you want to produce MOFs for, and can provide unique, per-node information. In your configuration script, you can then write logic - such as If constructs - that change the output of the MOF file for each node, based upon that configuration data. 

For example, you might create a configuration data block that, in part, looks like this:

```
$ConfData = 
@{
    AllNodes = 
    @(
        @{
            NodeName = "SERVER1";
            Role     = "WebServer"
        },
        @{
            NodeName = "SERVER2";
            Role     = "SQLServer"
        }
}
```

Then, in the configuration script, you might do something like this:

```
if ($node.role -eq "WebServer") {
	WindowsFeature InstallIIS {
		Ensure = 'Present'
		Name = 'Web-Server'
	}
}
```

The MOF files for each of those two nodes would therefore be different. Configuration data blocks can be defined in the same physical file as a configuration script, but more often the data is stored in a separate file, and then loaded and passed to the configuration script when that script is run.

While this technique does enable a single configuration script to handle a wide variety of situations, it also becomes possible for that configuration script to become incredibly complex - and potentially harder to maintain.

### Simple Includes
One way to make a complex configuration script easier to maintain is to break it into multiple files. Potentially, some of those chunks could then be reused across multiple configuration scripts. This isn't actually a complex technique; you just use standard PowerShell dot-sourcing. For example, consider the following as a sort of "master" configuration script:

```
configuration DatabaseServer {
	param(
		[string[]]$ComputerName
	)
	node $ComputerName {
		. ./standard-config.ps1
		. ./security-config.ps1
		. ./sql-server-2012-config.ps1
	}
}
```

When run, this configuration script would "include" (dot-source) three different PowerShell scripts. Those scripts would not contain the outer **configuration** block, nor would they contain a **node** block. Instead, they would simply contain configuration settings. As you can see, some of those three - like standard-config.ps1 or security-config.ps1 - might also be used in "master" configuration scripts that applied to different server roles.

This technique can also make use of configuration data blocks, enabling you to add per-node logic to each of the "included" scripts.

### Composite Configurations
In a composite configuration, you actually create a complete configuration script. It can even be set up to accept input parameters. For example, consider this extremely simple configuration script:

```
configuration customsyncfile {
	param(
		[string]$FilePath
	)
	node {
		SyncFile SyncFile1 {
			SourceFilePath = $FilePath
			DestFilePath = c:\inetpub\wwwroot
			SyncMode = 'Continuous'
			SyncMethod = 'FileHash'
			SyncTimeout = 5000
			SyncWait = 100
		}
	}
}
```

In this example, the configuration has been set up to accept a single input parameter, **FilePath**. Inside the configuration, I've referred to a (fictional) DSC resource named **SyncFile**. It obviously accepts a number of properties; I've provided values for each of these, but the **SourceFilePath** property will use whatever was given to the configuration's **FilePath** input parameter.

This configuration would then be saved in a way that _made it look like a DSC resource_ named "customsyncfile." This then becomes a _component configuration_. I would then write a second, _composite configuration_, perhaps like this:

```
configuration WebServer {
	param(
		[string[]]$ComputerName
	)
	node $ComputerName {
		
		customsyncfile WebServerFiles {
			FilePath = '\\dfsroot\webserverfiles\sources\source1'
		}
		
		WindowsFeature IIS {
			Name = 'Web-Server'
			Ensure = 'Present'
		}
		
	}
}
```

My composite configuration has two configuration settings, one of which is using the WindowsFeature DSC resource, and the other of which is using my **customfilesync** component resource. As you can see, I'm able to provide a **FilePath** to the component resource, which will be used as its **SourceFilePath** internally. I don't need to specify any other properties for **customfilesync**, because it wasn't set up to accept any additional input. Everything else that the underlying **SyncFile** DSC resource needs has already been hardcoded into the component configuration.

Composite configurations are a good way to modularize configuration elements. A component configuration can contain as much or as little as you need. You could, for example, have an entire component configuration that took care of every configuration element that your organization's IT Security team needed, while another one handled installation of standardized elements like management agents, encryption certificates, and so on. Each component configuration could be "owned" by a different team within your organization, helping to provide some separation of duties. 

Keep in mind that the component configurations do need to be present on the target nodes that use them - any DSC resources used _in_ the component configurations also need to be installed. Component configurations can, however, be ZIPped up and distributed by means of a Pull Server, just like any DSC resource module. 

I'll cover the technical details of composite configurations in a later chapter.

### Partial Configurations
Partial configurations are a new feature in PowerShell v5, and they're another way of modularizing configurations. They work best with a Pull Server setup, although it's possible to push partial configurations as well, and you can also have a mixed mode where you push some partials, and let the node pull other partials. 

So what's a partial configuration? Basically, you just create a bunch of different configuration scripts, run them to produce MOFs, and drop all the MOFs onto a Pull Server. You then configure a node's LCM to pull one or more of those MOFs. That configuration gets into some technical details that I'll cover later; for now, just understand that a single node can pull multiple MOFs. Some of those MOFs might be unique to that node, while others might be shared across many nodes.

When the LCM grabs all these MOFs, it first merges them into a single MOF. That's a bit more complex than just concatenating the files, but the LCM knows what to do. 

But a lot of us in the community have mixed feelings about partial configurations, and our trepidation comes from the uniqueness constraints I described earlier in this chapter. When you're authoring partial MOFs, it becomes really easily to _accidentally_ re-use the same configuration setting name. It also becomes easy for two separate MOFs to contain duplicate resource key properties. For example, MOF A might set the IIS WindowsFeature to install, while MOF B sets the IIS WindowsFeature to uninstall. You won't necessarily be aware of that as you sit on your authoring computer, but once the LCM merges the MOFs, it'll detect the conflicts and completely abort the configuration.

The problem we have with this is that the problem is occurring as far from you, the administrator, as possible. It's happening in the context of a background scheduled task, running on a remote computer. Debugging that situation - or even realizing that the LCM is exploding - can be hard. Right now, we don't have any way of _locally_ saying, "hey, suppose I send these 10 MOFs to a node - will any uniqueness problems pop up?" So there's no local "combine and test" or "what if" that you can run. 

Our feeling is that a smarter Pull Server is what's needed. A node could check in, and say, "hey, I need this list of 8 MOFs, please." The Pull Server could then load those off disk, _combine them on the server_, and send the merged MOF to the LCM. But the Pull Server could also detect any conflicts (and potentially even run some dynamic logic to alter the merged MOF based on criteria you provide). Any problems would be happening in one central place - the Pull Server - and could be logged, easier to get to, and easier to troubleshoot and solve. Sadly, right now, we don't have this kind of Pull Server.

In case you're still interested in partial configurations, I'll cover their technical details in an upcoming chapter.

### The Choice of Not Choosing
None of the above is meant to give you the impression that composites, includes, partials, or anything else are somehow mutually exclusive. They're not. You can have a partial configuration that uses a dot-sourced inclusion, and that is a composite configuration that relies on underlying component configurations. Any of those can have logic based on configuration data blocks. You can do it all.

### What's the Best Practice?
Well... other than the aforementioned concerns around partial configurations, there really _aren't_ best practices. None of these things is inherently good or bad. Different organizations work differently, have different requirements, and different preferences. DSC gives you different approaches that you can mix and match to help meet your unique situation.

## Understanding Dependencies
There's one more design thing you need to understand about configurations, and that's the order in which the LCM executes configuration settings from a MOF. Although today's LCM is single-threaded, Microsoft envisions a potential multi-threaded LCM someday. For that reason, you can't rely on the LCM simply executing configuration settings from the top of the MOF through to the bottom, as you might expect. But you will definitely have situations where you need such-and-such a configuration to finish before some other configuration. For example, you might want to copy an installer package to the node, and _then_ run the installer. Doing those in reverse won't work, and so you need some way of telling the LCM which one to do first. That way is called DependsOn.

```
WindowsFeature ActiveDirectoryFeature {
	Name = 'ADDS'
	Ensure = 'Present'
}

ADDomainController MyDC {
	Ensure = 'Present'
	DomainName = 'MyCompany'
	DCType = 'Regular'
	DependsOn = [WindowsFeature]ActiveDirectoryFeature
}
```

Here, I've specified a (fictional) DSC resource that sets up an AD domain controller. However, that configuration setting can't run until the WindowsFeature resource named "ActiveDirectoryFeature" has run. 

You can also specify that a configuration setting not run until _multiple_ other settings have run:

```
DependsOn = [WindowsFeature]IIS,[WindowsFeature]DNS
```

When the LCM executes its MOF, it will scan for these dependencies and sort the configuration settings accordingly. Note that there are some caveats when it comes to using DependsOn in a composite resource; I'll cover those in more detail in the chapter on composite configurations.