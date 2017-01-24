# Custom Resources
The time will come - probably quickly - when you need to do something with DSC that isn't supported by a resource that Microsoft or someone else has conveniently provided you. That's when you'll need to write a custom resource.

## Before We Begin: Function-Based vs. Class-Based
In WMF v5 and later, there are two types of resources that you can create. A *function-based resource*, which we've had since WMF v4, is just what the name implies: it's more or less a regular PowerShell script module, and it must contain three functions that have specific names and follow a specific usage pattern in their parameters. Get any of the specifics wrong, and it won't work. But aside from being careful about the naming, they're pretty straightforward, and they're what I'll be demonstrating in this chapter.

WMF v5 introduces *class-based resources*. Classes are an artifact of object-oriented programming. Honestly, from a code perspective, they're not all that different from functions. They look a little different, and you sort of lay out the code a little bit differently, but they're really similar. In object-oriented programming, one class can *inherit* from another class. So, suppose you have a base class called "vehicle" (all programming concepts can be illustrated through a vehicle analogy, it turns out). This "vehicle" class defines some of the common things all vehicles have, like a color, a model, and a manufacturer. The Vehicle class might also define some basic capabilities, like turning left or right, accelerating, decelerating, and so on. You might then code up a "car" class, which inherits from the Vehicle class. So the Car class magically has a color, model, and manufacturer property, because it inherited those from Vehicle. The Car class would also be *required* to implement capabilities like Turn, Accelerate, and Decelerate, because the base class demands that those be dealt with. Similarly, a Bicycle class would have to *implement* the same things, although its actual code for decelerating would obviously be different from that of the Car class.

So, classes in DSC resources inherit from a base class defined by Microsoft. This forces your class-based resource to have certain specific things, and editors like the PowerShell ISE can "check" you, to make sure you have those, before you even run the code. This helps eliminate the possibility of you mis-naming something, whereas in a function-based resource you're pretty much on the honor system to name things correctly. In other words, by using a class-based resource, you're less likely to screw up.

Professional developers love classes, and object-oriented programming, because in larger software projects a good OOP design can really make life easier. Code reuse is easier, screwing up is harder, and even debugging can be easier if you're doing it right. All my developer friends went nuts when PowerShell v5 added support for classes. Frankly, though, as an administrator, I was nonplussed. I always used snippets, for example, to start my function-based resources, and so I didn't care much about being "forced" to use the right naming conventions. My DSC resources (as you'll see) are extremely simplistic from a code perspective, and so classes just didn't seem to offer an advantage to me. But, whatever. I'm easygoing, and if you want to use classes, have fun.

But there's a downside in the current iteration of WMF v5, and it's a downside specifically for classes. You see, for a function-based resource, there's a special file naming convention when you ZIP up the resource for distribution to nodes. That convention lets the ZIP filename include the module's version number. Due to that convention, you can actually store multiple versions of the same module on the same pull server. Configuration scripts can specify which version of a module they want, and the nodes will grab the right version from the pull server. It's really nifty - it means you can deploy a revised module to a small number of nodes for testing, and then gradually roll it out simply by changing the configuration MOF that your nodes are pulling. Classes, however, don't presently support that versioning. It wasn't so much an oversight on Microsoft's part as it is the company's desire to "figure out versioning" in a much broader sense. Eventually, I'm told, we'll have proper versioning for side-by-side deployment of classes. Today, not so much - and for a lot of us in the DSC world, it's a deal-breaker for classes right now. Doing any kind of continuous integration, automated build pipeline is impossible without versioning. I don't want to plop a new revision of a resource onto a pull server, overwriting the known-working one, all at once. I want to roll it out on a schedule, like I can with function-based resources. So for me and a few of us in the crowd, class-based resources are still an interesting artifact that hasn't quite hit prime time. Your mileage may vary.

## Writing the Functional Code
In the next chapter, I'll be covering class-based resources, and after that I'll get into my best practices for resource design. For now, the short version is, "put all of your actual functionality into a normal PowerShell module, and just call those commands from the resource." The idea is that a resource, whether class- or function-based, is just an "interface." An adapter, if you will. Just as a GUI is an adapter between code and your eyeballs, a resource is an adapter between code and the LCM. When Microsoft wrote the Service resource, they didn't make low-level API calls from within the resource to start, stop, and configure services, right? No, the resource merely calls Start-Service, Set-Service, and so on. Those commands are external to the resource. And so now, we'll create a simple PowerShell module that will provide the functionality for an example resource.

Let me once again stress that the focus here is on _how to structure this_, not on how to write proper PowerShell functions and modules. If you're not already up on those practices, I heartily recommend *Learn PowerShell Toolmaking in a Month of Lunches* by myself and Jeffery Hicks.

```
Function Test-FileContent {
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory=$True)]
        [string]$Path,

        [Parameter(Mandatory=$True)]
        [string]$DesiredConent
    )

    if (Test-Path $Path) {
        # file exists; get it as a single string
        $ExistingContent = Get-Content -Path $Path | Out-String
        
        # compare
        if ($ExistingContent -ceq $DesiredConent) {
            Write $true
        } else {
            Write $false
        }

    } else {
        # file does not exist
        Write $false
    }

}

Function Set-FileContent {
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory=$True)]
        [string]$Path,

        [Parameter(Mandatory=$True)]
        [string]$DesiredConent
    )

    Set-Content -Path $Path -Value $DesiredConent

}
```

So, as you can see, I have a "Test" function that's comparing a file's contents to some desired set of content (yes, I realize the native File resource can do this, but the idea here is to make a simple example). I also have a "Set" function - and no "Get" function. I'm doing this deliberately to make a point, which will be explored in more detail in the "best practices" chapter coming up. Briefly, though, here's my philosophy: I want my actual resources to contain _as little code as possible_, because resources are harder to test and troubleshoot on their own. Ideally, I think of resources as just running one or two commands, and maybe applying a tiny bit of logic. Anything more complex than that, I separate into a "functional" module, which is just a normal PowerShell script module that I can code, test, and troubleshoot on its own.

In this case, I didn't include a Get-FileContent command because, frankly, it'd just be a thin wrapper around Get-Content. I know Get-Content already works, so I don't need to put a wrapper around it. In fact, my Set-FileContent function _is illustrating that point_, because it's nothing but a thin, thin, thin wrapper around Set-Content. Really, there's no need for my Set-FileContent - it isn't doing anything more than running a command that's already known to work. I included it here so that you could _see_ how minimal it is.

I would perhaps put this script module in my usual Modules folder. That way, when the LCM needs it, PowerShell will be able to find it. However, it's also possible to deploy the script module along with the resource module. I'll discuss that later in this chapter.

## Writing the Interface Module
What I'm calling the "interface module" is the actual DSC resource. There are some requirements for function-based resources that are a bit hard to create manually, such as the schema MOF. I'll get to all of that - and an easier way to do it - in a bit; for right now, let's just look at the code.

```
function Get-TargetResource {
    [CmdletBinding()]
    [OutputType([System.Collections.Hashtable])]
    Param(
        [Parameter(Mandatory=$True)]
        [string]$Path
    )

    If (Test-Path $Path) {
        $content = Get-Content -Path $path | Out-String
    } else {
        $content = $null
    }

    $output = @{
        Path = $Path
        Content = $content
    }
    Write $output
}

function Test-TargetResource {
    [CmdletBinding()]
    [OutputType([System.Boolean])]
    Param(
        [Parameter(Mandatory=$True)]
        [string]$Path,

        [Parameter(Mandatory=$True)]
        [string]$DesiredContent
    )

    Write Test-FileContent -Path $Path -DesiredContent $DesiredContent

}

function Set-TargetResource {

    [CmdletBinding()]
    Param(
        [Parameter(Mandatory=$True)]
        [string]$Path,

        [Parameter(Mandatory=$True)]
        [string]$DesiredContent
    )

    Set-FileContent -Path $Path -Content $Content

}
```

Notice that my script module includes three functions, which have specific names I must use: Test-, Get-, and Set-TargetResource. The LCM only knows to call these names, and you must use those names, although the functions can appear in any order within your module. 

In the Get- function, notice that I'm returning a hashtable that contains whatever configuration currently exists. You provide the path, and I'll get you the file content. This is used when you run Get-DscConfiguration. 

Next is the Test- function, which _uses the function I created earlier_ and returns either True or False. This is used when you run either Start-DscConfiguration or Test-DscConfiguration, and of course when running a consistency check. If the LCM is configured to autocorrect, then Test- returning a False will cause the LCM to immediately run Set-. 

Set- illustrated how pointless my Set-FileContent function is, since it isn't any more complex than just running the underlying Set-Content directly.

There's an important procedural point, here: if the LCM runs Test, and if Test returns False, then the LCM may (if configured) run Set. _The LCM assumes the Set to have been successful if the Set function does not throw an error_. That is, the LCM doesn't run Set, and then run Test again to verify. It simply runs Set. So it's important to write a _good_ Set function, one that can return errors if it fails. In this case, I'm not trapping any errors, so if one occurs - like a permission denied, or "Path does not exist," or something, then that error will be thrown, and the LCM will "see" it and know that the Set didn't work.

So this is now a minimally functional DSC resource.

## Preparing the Module for Use and Deployment
First, you need to make sure that your DSC resource is in the right file structure. **This applies to the DSC resource, not to the "functional module" that I created first**. That functional module gets stored just like any other PowerShell script module. Well, with an optional exception that I'll get to in a sec. The base folder for a resource module needs to go into one of the folders listed in the PSModulePath environment variable, like /Program Files/WindowsPowerShell/modules. For example, suppose my DSC resource module is named MyDSCStuff, and I've written a function-based resource named AppControl, which will live within that resource module. I've also written a second resource named AppUsers, which will live in the same resource module, so that they're distributed as a set.

Note that you should **not** store resources in your Documents folder. The LCM runs as SYSTEM, and won't be able to find stuff in Documents. Use the /Program Files location unless your PSModulePath defines some other globally-accessible location. A UNC also won't work, as SYSTEM can't normally access non-local files.

* /Program Files/WindowsPowerShell/Modules/MyDSCStuff/MyDSCStuff.psd1 is the "root" for the resource module. You can run New-ModuleManifest to create MyDSCStuff.psd1. 
* /Program Files/WindowsPowerShell/Modules/MyDSCStuff/DSCResources is the folder where the individual resources will live.
* /Program Files/WindowsPowerShell/Modules/MyDSCStuff/DSCResources/AppControl/AppControl.psd1 and AppControl.psm1 are the function-based resource AppControl. The .psm1 file contains my three -TargetResource functions, and the .psd1 should simply define the .psm1 file as the root module, and specify the three -TargetResource functions to be exported (technically, the .psd1 is optional, so you can omit it). You also need AppControl.schema.mof, which we'll come to in a moment.
* /Program Files/WindowsPowerShell/Modules/MyDSCStuff/DSCResources/AppUser/AppUser.psd1 and AppUser.psm1 are the function-based resource AppUser. You also need AppUser.schema.mof, which we'll come to in a moment.

The .psd1 files are always a good idea to include, because they can speed up certain background operations PowerShell has to perform, especially when the .psd1 file explicitly exports the three -TargetResource functions. This isn't as much a huge deal as it is for normal PowerShell modules, but having a manifest - the .psd1 file - is a good habit to be in.

The .schema.mof file is essential for each resource. And... um, you kind of have to build it by hand. Normally. Fortunately, Microsoft has a Resource Designer that can actually make the schema MOF for you, in addition to setting up all the right files and folders for your resource. Check out <https://msdn.microsoft.com/en-us/powershell/dsc/authoringresourcemofdesigner>. It's a set of commands that can build out the necessary files and folders, as well as the right function names, all for you. 

Now... I mentioned an optional exception for dealing with your "functional" module, remember? So far, I've told you that your functional module needs to be a normal PowerShell script module. It lives apart from the DSC resource, meaning you'll have to distribute it independently. That might be fine with you - if you're smart, and you already have a good module-deployment mechanism in place (like a private NuGet repository or PowerShell Gallery instance), then it's no big deal. But you can _also_ include the functional module with the DSC resource module, since we're using function-based resources. Following the above folder structure example, /Program Files/WindowsPowerShell/Modules/MyDSCStuff/MyDSCStuff.psm1 would be the filename for your "functional" module. MyDSCStuff.psd1 should define that .psm1 as the root module, and should export whatever functions the module contains. With this approach, the functional module will still be an operationally independent entity, but it'll be "packaged" with the DSC resources.

So... with the right folder structure in place, just ZIP it all up. You should be creating MyDSCStuff.zip (in this example), and when you unZIP it, it should recreate the correct folder structure. And finally, change the filename to include a version number, such as MyDSCStuff_1.2.zip. Then, create a checksum file using New-DscChecksum, so that you also have MyDSCStuff_1.2.zip.checksum. Those two files go to your pull server's Resources folder, and your nodes will be able to find them there.

## Triggering a Reboot
All resources run within a PowerShell scope created by the LCM. When it spins up each new scope, the LCM creates a global variable named $global:DSCMachineStatus. Setting this variable to 1, from anywhere in your resource's Set code, will tell the LCM that a reboot is needed. The LCM configuration's RebootNodeIfNeeded setting will determine if the node is rebooted immediately or not.

It's important to understand that this behavior only works within your Set code - a Get or Test cannot trigger a reboot. Further, your Set must complete without error (which is how the LCM knows it was successful).
