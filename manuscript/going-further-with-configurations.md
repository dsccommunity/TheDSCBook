# Going Further with Configurations
To this point, we've basically treated configurations and nodes as a 1:1 thing. One configuration script gets you one MOF, which goes to one node. For some environments, that'll be totally fine. For others, you'll want a bit more modularization and re-use. After all, many nodes will be similar, so why copy-and-paste all that script code?

## Again: DSC isn't Tooling
I feel it's important at this stage to remind you that DSC is a platform, not a solution set. That is, there's no tooling. _Ideally_, many of us want some kind of System Center-y thing, where we can assign "roles" to our nodes, and have some magical back-end database spew out the necessary MOF files directly to a pull server. That'd be great, and I bet we'll get it one day. But that isn't what this book is about, anyway. So we're going to stick with what's "in the box," and you'll likely end up using a mix-and-match of the techniques I'm covering to meet your needs.

## Understanding ConfigurationData
One of the first ways that you can have a single configuration script produce multiple, unique MOFs, is to supply the script with _configuration data_. This is something that lives _apart from_ the script itself, and is fed to the script when the script is run. So, just to set the stage broadly, a script file might look like this:

```
configuration MyMasterConfig {
  # a bunch of stuff goes here
}

$configData = @{}

MyMasterConfig -ConfigurationData $configData
```

I've created a configuration named **MyMasterConfig**. Separately - and this could have been in a totally different file that I dot-sourced, for example - I created a $configData hash table. I'm not showing you the structure of that yet, but I will in a bit. Then, I run the config, and pass in the hash table using a built-in parameter. Now, my configuration data is available "inside" MyMasterConfig.

Let's start digging into the actual details.

## Defining Configuration Data
The top level configuration data hash table needs to contain two elements, AllNodes and NonNodeData. AllNodes is an array, and it'll contain all of the unique-to-each-node information that you need to provide. NonNodeData is a kind of "global" section, and we'll see later how it's used.

I mentioned that AllNodes is an array. Each element within the array is a new hash table, which must at least contain a NodeName element. Each can then contain whatever other elements you want, and you basically get to make them up - there are no requirements or specifics. So, as a starting point:

```
$MyData =
@{
    AllNodes =
    @(
        @{
            NodeName = 'NODE1'
            Role     = 'WebServer'
        },
 
        @{
            NodeName = 'SERVER2'
            Role     = 'WordPress'
        },
 
        @{
            NodeName = 'MACHINE3'
            Role     = 'WebServer'
        }
    );
    NonNodeData = ''  
}
```

Here, I've defined three nodes, named NODE1, SERVER2, and MACHINE3. For each, I've defined a "Role" property. That's a property I made up; DSC doesn't care what other properties I provide. I've specified a value for each Role, and again, this is data that's only meaningful to me. This configuration data isn't "connected" to any configuration script at this point. In order to create that connection, I would run the configuration, and use its built-in -ConfigurationData parameter to pass in $MyData. You do not need to define -ConfigurationData in a Param() block; it's a magic parameter that is automatically available for all configuration scripts. 

## Referencing and Using Configuration Data
Okay, so you've created a data block, and you've passed it into a configuration. How do you use it? Inside your configuration - and remember, you've passed the Configuration Data _into_ the configuration script by means of the -ConfigurationData parameter - you'll use three built-in variables:

* $AllNodes automatically gets you the AllNodes@() portion of your ConfigurationData block. In the next section, I'll share some of its super powers. One of those powers is the ability to filter down the nodes - for example, only grabbing nodes that have a certain property set to a certain value. 
* Once you've filtered the $AllNodes collection, you can use $Node to refer to just the current node.
* $ConfigurationData represents the entire block of data.

So let's see a bit of that in action. Remember that, in a configuration script, you start out with a Node block. That's where you can use $AllNodes, and it does a pretty impressive bit of magic. Pretend that the above three-node data block has been defined in the same script as this example:

```
configuration AllMyServers {
  Node $AllNodes.Where({$_.Role -eq "WebServer"}).NodeName {
  }
}

AllMyServers -ConfigurationData $MyData
```

Microsoft uses this example a lot, but rarely explains the magic (there are also a lot of older examples that have the wrong syntax). The Node{} construct wants you to provide a list of node names. That's it - just computer names, and no other data. So we're taking $AllNodes, which contains everything. We're using its special Where method to grab only those nodes whose Role property is set to WebServer. Remember that _I made up_ the Role property, right? So I really could have filtered on any property that I'd put into the data block. Having done that filtering, I'm left with two nodes. Each node has a NodeName and a Role property - but the Node{} block here in the script _only wants the names_. So that's why the ".NodeName" is there - it takes the filtered nodes and grabs just the _contents_ of the NodeName property.

Now let's add a bit more:

```
configuration AllMyServers {
  Node $AllNodes.Where({$_.Role -eq "WebServer"}).NodeName {
    xWebSite TheWeb {
      Name = $Node.NodeName
      PhysicalPath = 'c:\inetpub\wwwroot'
      Ensure = 'Present'
    }
  }
}

AllMyServers -ConfigurationData $MyData
```

Within the Node{} construct, PowerShell will automatically run the contents once for each node you've given it. It's kind of like an invisible ForEach wrapped around the whole thing. Inside the Node{} construct, I can use $Node to refer to "the current node." So on the first pass, $Node will refer to NODE1. On the second pass, it'll refer to MACHINE3. Go back and look at my data block if you need to remind yourself why.

You can see that this is going to produce two MOF files, one named NODE1.mof and another named MACHINE3.mof, both (by default) in a folder named /AllMyServers. Each MOF will define a website, which will have the same name as the server itself. That's actually not realistic, so let's revisit my data block and add some stuff to it:

```
$MyData =
@{
    AllNodes =
    @(
        @{
            NodeName = 'NODE1'
            Role     = 'WebServer'
            Site     = 'CustomApp'
            SitePath = 'c:\inetpub\approot'
        },
 
        @{
            NodeName = 'SERVER2'
            Role     = 'WordPress'
        },
 
        @{
            NodeName = 'MACHINE3'
            Role     = 'WebServer'
            Site     = 'App2'
            SitePath = 'c:\inetpub\app2root'
        }
    );
    NonNodeData = '' 
}
```

Notice that I've added Site and SitePath properties only to the nodes that have a WebServer value for Role. That's totally legal - you do not need to provide consistent properties for each node. You simply have to make sure you're providing whatever properties your configuration script is expecting. So now I can use those two new properties:

```
configuration AllMyServers {
  Node $AllNodes.Where({$_.Role -eq "WebServer"}).NodeName {
    xWebSite TheWeb {
      Name = $Node.Site
      PhysicalPath = $Node.SitePath
      Ensure = 'Present'
    }
  }
}

AllMyServers -ConfigurationData $MyData
```

You can see how the data block allows me to use a single configuration script to produce multiple, unique MOFs for multiple nodes.

## All-Nodes Data
You can also specify properties to apply to all nodes:

```
$MyData =
@{
    AllNodes =
    @(
        @{  NodeName = '*'
            SystemRoot = 'c:\windows\system32'
         },
         
        @{
            NodeName = 'NODE1'
            Role     = 'WebServer'
            Site     = 'CustomApp'
            SitePath = 'c:\inetpub\approot'
        },
 
        @{
            NodeName = 'SERVER2'
            Role     = 'WordPress'
        },
 
        @{
            NodeName = 'MACHINE3'
            Role     = 'WebServer'
            Site     = 'App2'
            SitePath = 'c:\inetpub\app2root'
        }
    );
    NonNodeData = '' 
}
```

Now, all three of my nodes also have a SystemRoot property. This is exactly the same as if I'd manually listed SystemRoot for each node individually, but it saves some room. The * is a special character in this case; don't think of it as a usual wildcard, because other wildcard expressions aren't allowed.

## Using the $AllNodes Variable
The $AllNodes variable, then, becomes a key for making those MOFs unique. It has two special powers, one of which you've seen: Where and ForEach. You've seen how Where can be used to select just a subset of nodes. For example, using my most recent $MyData data block example, all of the following are legal:

```
 $AllNodes.Where({ $_.Site -eq 'CustomApp' }) # returns 1 node
 $AllNodes.Where({ $_.NodeName -like 'NODE*' }) # returns 1 node
 $AllNodes.Where({ $_.SystemRoot -eq 'c:\windows\system32' }) # returns 3 nodes
```

I also mentioned the ForEach() method of $AllNodes. Frankly, I don't find myself using that for nodes, because the Node{} construct is already an implicit ForEach. However, there are certainly times when it can be useful. Imagine this configuration data block:

```
{
AllNodes = @(
@{
  NodeName = '*'
  FilesToCopy = @(
    @{
      SourcePath = 'C:\SampleConfig.xml'
      TargetPath = 'C:\SampleCode\SampleConfig.xml'
    },
    @{
      SourcePath = 'C:\SampleConfig2.xml'
      TargetPath = 'C:\SampleCode\SampleConfig2.xml'
    }
}
```

Now you might use a ForEach loop:

```
configuration CopyStuff {

  Node $AllNodes.NodeName {
  
    $filenumber = 1
    ForEach ($file in $Node.FilesToCopy) {
      File "File$filenumber" {
        Ensure = 'Present'
        Type = 'File'
        SourcePath = $file.SourcePath
        DestinationPath = $file.TargetPath
      }
      $filenumber++
    }
  
  }

}
```

Except I didn't actually use $AllNodes.ForEach(), right (grin)? That's just a plain ForEach loop, but it does show how it can sometimes be beneficial to do that. Honestly... I really do struggle to find use cases for $AllNodes.ForEach(). I've always found it simpler to filter using Where(), and to just use a normal ForEach loop - as above - when I need to enumerate something from the data block.

## Configuration Script Strategies
If you were really paying attention to all of the foregoing examples, you might have noticed that I only generated MOFs for NODE1 and MACHINE3. What about SERVER2?

It depends.

### Option 1: Multiple Configuration Scripts
One option is to keep your configuration data block in a separate file, and then create multiple configuration scripts for different kinds of nodes. That same block can be fed to multiple configuration scripts. For example:

```
$MyData = @{
  $AllNodes = @(... data goes here ...)
  $NonNodeData = ''
}

. ./Config1.ps1 # Contains "MyFirstConfig" configuration
. ./Config2.ps1 # Contains "MySecondConfig" configuration

MyFirstConfig -ConfigurationData $MyData
MySecondConfig -ConfigurationData $MyData
```

So I've got two configurations, which I've put into separate script files for my convenience. I've dot-sourced those into the same script that contains my data block, and then passed the same data block to each. Presumably, each configuration is filtering the nodes in some fashion, so they're not necessarily duplicating each other's efforts. Obviously, I haven't completely filled in the $AllNodes section with actual node properties - I was just trying to lay out a concise framework to illustrate the approach.

### Option 2: Multiple Node{} Blocks
Very similar to Option 1 is to simply include more than one node block in a configuration script.

```
configuration UberConfig {
  Node $AllNodes.Where({ $_.Role -eq 'WebServer' }).NodeName {
  }

  Node $AllNodes.Where({ $_.Role -eq 'WordPress' }).NodeName {
  }
}
```

That's totally legal, and running UberConfig with my $MyData block would produce three MOFs - one for NODE1, one for SERVER2, and one for MACHINE3. 

### Option 3: Lots of Logic
You can also take a more complex approach where you _don't_ filter the nodes in the Node{} construct, and instead use a lot of logic constructs.

```
configuration LogicalConfig {
  Node $AllNodes.NodeName {

    if ($Node.Role -eq "WebServer") {
      # add settings here
    }
    
    if ($Node.Role -eq "WordPress") {
      # add settings here
    }

  }
}
```

This gives you a more declarative kind of control over what goes into each MOF, but it can obviously get harder to read and maintain if you go nuts with it. 

## Using NonNodeData
There's a couple of important things to know about this sometimes-misused section.  First, you do not have to call the section "NonNodeData", and second, you can have multiple NonNodeData sections, which makes a NonNodeData section a good option for role-specific settings that do not pertain to all nodes.  The snippet below shows two NonNodeData sections, one for domain controller settings (DCData) and the other for DHCP server settings (DHCPData).

```
$MyData = @{
  AllNodes = @( ... )
  
  DCData = @{
            DomainName = "Test.Pri"
            DomainDN = "DC=Test,DC=Pri"
            DCDatabasePath = "C:\NTDS"
            DCLogPath = "C:\NTDS"
            SysvolPath = "C:\Sysvol"
            }

  DHCPData = @{
            DHCPName = 'DHCP1'
            DHCPIPStartRange = '192.168.3.200'
            DHCPIPEndRange = '192.168.3.250'
            DHCPSubnetMask = '255.255.255.0'
            DHCPState = 'Active'
            DHCPAddressFamily = 'IPv4'
            DHCPLeaseDuration = '00:08:00'
            DHCPScopeID = '192.168.3.0'
            DHCPDnsServerIPAddress = '192.168.3.10'
            DHCPRouter = '192.168.3.1'
            } 
}
```

Keep in mind that you have the special $ConfigurationData variable, and $ConfigurationData.<NonNodeData> will access the "global" NonNodeData keys.   Previously, we'd just set NonNodeData to an empty string (""), but in fact it can be a hash table or multiple hash tables. So now, the configuration can use $ConfigurationData.DCData.DomainName to access the domain name property, or $DHCPData.DHCPDNSServerIPAddress to access the DHCP server's DNS IP address. 