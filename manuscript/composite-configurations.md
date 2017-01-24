# Composite Configurations
Composite configurations are one way to modularize commonly used groups of configuration settings. You start by building one or more "sub-configurations." That isn't an official term, but rather one I made up and find useful. A sub-configuration contains that group of commonly used configuration settings. It can be parameterized, which means it can slightly alter its output based on the input values you feed it. You then save that sub-configuration in a special way _that makes it look like a DSC resource_. 

Next, you write the actual _composite_ configuration. This is just a normal configuration that uses both regular DSC resources, and the looks-like-a-resource sub-configurations. 

The official term for sub-resources is a _composite resource_, because it can take multiple DSC resources and merge, or "composite", them together into a single looks-like-a-resource. Technically, I don't think Microsoft is using the term "composite configurations" anymore, and "composite resource" is more accurate. But "composite configuration" has been thrown around since DSC was invented, so I'm gonna run with it.

I'm going to run through an example that uses code from https://msdn.microsoft.com/en-us/powershell/dsc/authoringresourcecomposite, and add my own explanations and ideas as I go. 

## Creating a Composite Resource
Start by writing a perfectly normal DSC resource. You can actually run this on its own, produce a MOF, and test it. Here's the Microsoft example:

```
Configuration xVirtualMachine
{
    param
    (
        # Name of VMs
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String[]] $VMName,

        # Name of Switch to create
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $SwitchName,

        # Type of Switch to create
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $SwitchType,

        # Source Path for VHD
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $VHDParentPath,

        # Destination path for diff VHD
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $VHDPath,

        # Startup Memory for VM
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $VMStartupMemory,

        # State of the VM
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [String] $VMState
    )

    # Import the module that defines custom resources
    Import-DscResource -Module xComputerManagement,xHyper-V

    # Install the Hyper-V role
    WindowsFeature HyperV
    {
        Ensure = "Present"
        Name = "Hyper-V"
    }

    # Create the virtual switch
    xVMSwitch $SwitchName
    {
        Ensure = "Present"
        Name = $SwitchName
        Type = $SwitchType
        DependsOn = "[WindowsFeature]HyperV"
    }

    # Check for Parent VHD file
    File ParentVHDFile
    {
        Ensure = "Present"
        DestinationPath = $VHDParentPath
        Type = "File"
        DependsOn = "[WindowsFeature]HyperV"
    }

    # Check the destination VHD folder
    File VHDFolder
    {
        Ensure = "Present"
        DestinationPath = $VHDPath
        Type = "Directory"
        DependsOn = "[File]ParentVHDFile"
    }

    # Create VM specific diff VHD
    foreach ($Name in $VMName)
    {
        xVHD "VHD$Name"
        {
            Ensure = "Present"
            Name = $Name
            Path = $VHDPath
            ParentPath = $VHDParentPath
            DependsOn = @("[WindowsFeature]HyperV",
                          "[File]VHDFolder")
        }
    }

    # Create VM using the above VHD
    foreach($Name in $VMName)
    {
        xVMHyperV "VMachine$Name"
        {
            Ensure = "Present"
            Name = $Name
            VhDPath = (Join-Path -Path $VHDPath -ChildPath $Name)
            SwitchName = $SwitchName
            StartupMemory = $VMStartupMemory
            State = $VMState
            MACAddress = $MACAddress
            WaitForIP = $true
            DependsOn = @("[WindowsFeature]HyperV",
                          "[xVHD]VHD$Name")
        }
    }
}
```

Notice the huge Param() block right at the beginning. It wants the name of a virtual machine (VM), the name of a virtual switch, the switch type, a couple of paths, startup memory size, and the state of a VM. Basically, all the core things you need to define a new virtual machine. You'll notice those parameters - $SwitchName, $Name, and so on - being used throughout the rest of the configuration. 

## Turning the Configuration into a Resource Module
This gets saved into a special directory structure. It needs to start in one of the folders in the PSModulePath environment variable, such as /Program Files/WindowsPowerShell/modules. So, let's say we wanted this thing called MyVMMaker - that'll be the resource name we use to refer to this thing from within another configuration. We want the overall resource _module_ to be called MyVMStuff. Remember, a _module_ can contain multiple _resources_. I'll save:

/Program Files/WindowsPowerShell/modules/MyVMStuff/MyVMStuff.psd1
/Program Files/WindowsPowerShell/modules/MyVMStuff/DSCResources/MyVMMaker.psd1
/Program Files/WindowsPowerShell/modules/MyVMStuff/DSCResources/MyVMMaker.schema.psm1

So I need three files. MyVMMaker.schema.psm1 is the configuration that I wrote above. The other two files are _manifests_, and I need to create those. MyVMMaker.psd1 only needs one line:

```
RootModule = 'MyVMMMaker.schema.psm1'
```

To create the other .psd1, just run:

```
New-ModuleManifest -Path \Program Files\WindowsPowerShell\modules\MyVMStuff\MyVMStuff.psd1
```

What's even easier is to grab the helper function from https://blogs.technet.microsoft.com/ashleymcglone/2015/02/25/helper-function-to-create-a-powershell-dsc-composite-resource/, which includes a New-DSCCompositeResource function. It'll automatically set up the correct folder structure, .psd1 files, and template schema.psm1 file you can use as a starting point.

## Using the Composite Resource
Now I can refer to the composite resource in a normal configuration:

```
configuration RenameVM
{

    Import-DscResource -Module MyVMStuff
    Node SERVER1
    {
        MyVMMaker NewVM
        {
            VMName = "Test"
            SwitchName = "Internal"
            SwitchType = "Internal"
            VhdParentPath = "C:\Demo\VHD\RTM.vhd"
            VHDPath = "C:\Demo\VHD"
            VMStartupMemory = 1024MB
            VMState = "Running"
        }
    }

}
```

You can see where I've explicitly imported the DSC resource _module_ MyVMStuff, and referred to the MyVMMaker _resource_, passing in all the settings defined in the composite resource's Param() block.

## Deploying the Composite Resource
To deploy the composite resource to a pull server, ZIP it up. Basically, in my example, you'd ZIP up the MyVMStuff folder under /Program Files/WindowsPowerShell/Modules, producing /Program Files/WindowsPowerShell/Modules/MyVMStuff.zip. Rename that to include a version number, like MyVMStuff_1.0.zip.

Then, create a checksum file by running:

```
New-DscCheckSum -Path MyVMStuff_1.0.zip -force. 
```

Both the ZIP file and the checksum file would go on the pull server, in its designated Resources path. If you plan to manually deploy the resource, you don't need to do any of this - you can just get the correct folder hierarchy, and the files, in place in whatever fashion you desire.

## Approach Analysis
As I wrote in the previous chapter, every modularization approach has pros and cons.

* CON: Like any other DSC resource you use that doesn't ship with the operating system, you have to deploy your composite resource. That can mean ZIPping it up, creating a checksum for it, and deploying it to a pull server. Or, it might mean manually deploying it to nodes, if you're not using a pull server.
* PRO: The "contents" of the composite resource are distinct from the configuration that calls it. So things like duplicate keys aren't a problem in the way that they can be for a Partial configuration.
* PRO: It's a little easier to see your entire configuration "in one place," because while the composite resource itself is separate, your "main" configuration _shows what the composite resource is doing_. That is, in my example, you can see that it's creating a VM, what the VM name is, and so on. Now, it's certainly possible to have a composite resource with a bunch of hardcoded information that won't be "seen" in the main configuration - but that's your decision to take that approach.
* PRO: Composite configurations are very structured. 
* CON: There are some limitations on using DependsOn within composite resources. From the main configuration, any setting can have a DependsOn for any other setting _in the main configuration_. But you can't take a DependsOn to _an individual setting within the composite resource_. If you're designing composite resources well, you can minimize this as a problem. Simply keep each composite resource as "atomic" as possible, meaning whatever it's doing should be tightly coupled and not designed to be separated.
* PRO: It's relatively easy to test a composite resource, by simply building a "main" configuration that uses the composite resource and nothing else. You can also just "run" the composite resource standalone, because, after all, _it's just a configuration script_. Supply whatever parameters it wants, produce a MOF, and deploy that MOF to a test machine to validate it.

## Design Considerations
As I noted above, the key to designing composite resources is to have them do one thing, and one thing only. Whatever's happening inside any given resource should all be intended to happen together, or not at all. For example, Microsoft's sample - which I used - creates a VM. You'd never create a VM without assigning memory, setting up a virtual disk, and so on, and so everything in that resource _goes together_. 

You might have a set of security configurations - firewall stuff, maybe some anti-malware packages, and Windows Update settings - that you always want applied as a collection to your nodes. That's fine - that might make a good resource. What you want to avoid, however, are monolithic composite resources that perform several unrelated tasks, like making a machine a domain controller _and_ installing DHCP _and_ installing something else. Those might all be common elements of your organization's "infrastructure servers," and each DC and DHCP server might be configured exactly the same, but here's how I'd handle it:

* A composite resource that sets up a company-standard DC
* A composite resource that sets up a company-standard DHCP server
* A "main" configuration that calls the DC and DHCP composite resources, and also sets up any node-unique information like computer name. 

The idea is to keep each composite resource as tightly scoped as possible, and let the "main" configuration bring them together into a complete node config.