# Finding and Using Resources
At this point in the book, I feel that we've gotten all the basics out of the way. You should know who the major players are in the DSC process, and you should have all the core terminology in your head. So now we can get serious.

I think the first thing people tend to ask about DSC is, "what can I do with it?" Legit question. And although I wanted to make sure you knew about configurations, MOFs, the LCM, and all that, for me, "what can I do with it?" is the right starting point for DSC. Figuring out what DSC resources are available, and figuring out how to use them, is pretty much where DSC really begins.

One major upside to DSC is that so much is available as open source projects. Nearly everything Microsoft's created for DSC is open source. Tons of open source projects from community groups and enthusiasts are out there. And many of those projects have dozens (or more!) of contributors, all improving and debugging everything, all the time. Things move and improve fast - which is great. But the corresponding downside is that documentation can be slim or nonexistent. So figuring out how to use what's out there - that's critical.

## Finding What's Out There
Let's be clear: there is no way I, or anyone, can document all the open source DSC resources out there. What I can do, however, is point you to some of the major locations where you'll find them. 

First, use the [PowerShell Gallery](http://powershellgallery.com). This is a huge repository of PowerShell modules, including DSC resources. These are usually the "production ready" modules that you want to use. It's meant to be accessed from within PowerShell itself, using PowerShellGet and PowerShell Package Manager. The site contains download links for PowerShell Package Manager in the event you don't have it, although it's a native part of WMF5, so you should. Using the Find-Module command within PowerShell, you can even limit your results to just DSC resources. There are actually a few ways to do that filtering, here's the one I use a lot:

```
PS C:\> find-module -tag dsc

Version    Name                                Repository           Description
-------    ----                                ----------           -----------
1.11.0.0   xWebAdministration                  PSGallery            Module with DSC Resources for Web Administration
3.10.0.0   xPSDesiredStateConfiguration        PSGallery            The xPSDesiredStateConfiguration module is a part...
2.9.0.0    xNetworking                         PSGallery            Module with DSC Resources for Networking area
2.11.0.0   xActiveDirectory                    PSGallery            The xActiveDirectory module is originally part of...
1.9.0.0    xDSCResourceDesigner                PSGallery            This module is meant to assist with the developme...
1.6.0.0    xComputerManagement                 PSGallery            The xComputerManagement module is originally part...
1.6.0.0    xSQLServer                          PSGallery            Module with DSC Resources for dep
```

I've truncated those results, by the way - you can run the command yourself to see them all. 

**Note** - You may still see online references to the "DSC Resource Kit." That was before Microsoft moved to PowerShell Gallery; you should rely on what's in the Gallery rather than downloading the old Kit.

_Many_ of the DSC resource authors in the community publish their modules to PowerShell Gallery. If you happen to run across their source code repositories - often GitHub - you should use what's in Gallery rather than downloading the code directly. Often times, what's in Gallery represents a "stable" release, while what's in the source code repo may be a "development branch."

And aside from resources authored by Microsoft (the full output of Find-Module can tell you that), you should treat any code in Gallery as untrusted. That is, you should review it and be comfortable with it before deploying it in your production environment.

Not _everyone_ publishes to the Gallery, though, and so Google remains one of your best friends when it comes to finding new DSC resources.

## Installing What's Out There
OK: you've found the awesomeness you need online. How do you install it?

If the DSC resource module you're after happens to be in an online gallery - like PowerShell Gallery - simply use the Install-Module command to install it to your local computer. Note that this will _not_ necessarily give you something you can immediately deploy to all the target nodes that will need it; right now, we're just getting the module onto your authoring machine. We'll worry about deploying it to other nodes later.

If the module you need isn't in a Gallery that Install-Module can get to, then you're likely just downloading a ZIP file. That's fine. Unzip it to your /Program Files/WindowsPowerShell/Modules folder. Now, be a little careful, here. Some folks will ZIP up _that entire path_, meaning you'll end up with /Program Files/WindowsPowerShell/Modules/Program Files/WindowsPowerShell/Modules/WhateverTheModuleNameIs. That won't work. Supposing the module is named Fred (for some reason), you ultimately want /Program Files/WindowsPowerShell/Modules/Fred. Inside the /Fred folder, all of the module's files should exist, and that may include one or more subfolders, too.

## Finding What's Installed
Here's the test: making sure PowerShell can actually "see" the modules you've installed. This is super-important, because if this step fails, then nothing will work. So to test that, run Get-DscResource.

```
ImplementedAs   Name                      ModuleName                     Version    Properties
-------------   ----                      ----------                     -------    ----------
Binary          File                                                                {Destina...
PowerShell      Archive                   PSDesiredStateConfiguration    1.1        {Destina...
PowerShell      Environment               PSDesiredStateConfiguration    1.1        {Name, D...
PowerShell      Group                     PSDesiredStateConfiguration    1.1        {GroupNa...
```

As you can see, I have several resources - Archive, Environment, and so on - in a _resource module_ called PSDesiredStateConfiguration, and it's version is 1.1. After running Get-DscResource, you should see any resource modules you've installed; if you don't, _get that sorted out before you go any further_. 

The main problem I've seen is simply having the resource module in the wrong location. Have a look at the PSModulePath environment variable on your computer - it should include the /Program Files/WindowsPowerShell/Modules folder as one of its locations. If it doesn't, fix it (which needs to be done outside PowerShell, like from the System Properties Control Panel applet). Or, locate your modules in one of the locations that PSModulePath _does_ contain (ideally not the System32 location; that's meant for Microsoft only). 

## Figuring Out What a Resource Wants
Once a resource is installed and PowerShell can find it, you're ready to start figuring out how to use it. In the earlier chapter on authoring basic configurations, I noted that a configuration setting looks something like this:

```
ResourceName UniqueName {
    Property = Value
    Property = Value
}
```

The ResourceName is easy to figure out, because it's shown right in the output of Get-DscResource. But what properties does the resource support? What values do they accept? _Hopefully_ there's some documentation out there that can clue you in! Microsoft's gotten better about publishing documentation for a lot of their resources, although as I write this they're still working on adding more docs. But don't rely on the Get-Help command in PowerShell - at this time, it's not wired up to provide help on DSC resource modules, unfortunately. Well, not directly.

Have a look at PS C:/Windows/System32/WindowsPowerShell/v1.0/Modules/PSDesiredStateConfiguration/en-US on your computer. You should see a whole bunch of help files. Some are for commands like Disable-DscDebug, and you can use the ordinary Help command to view that documentation. Some of the rest of it is a little harder, because you can't just run "Help PSDesiredStateConfiguration" and get that documentation. What you can run, however, is this:

```
PS C:\Windows\System32\WindowsPowerShell\v1.0\Modules\PSDesiredStateConfiguration\en-US> help -path .\PSDesiredStateConfiguration.psm1-help.xml
```

That is, by using the -Path parameter and providing the path of the XML help file, you can read the documentation right in the shell. So that's definitely a starting point. Now, let's use that to get resource-specific. Keeping with the same PSDesiredStateConfiguration module, if I look at its root directory (that is, a level up from en-US), I see these entries (amongst others):

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        7/10/2015   4:04 AM                DownloadManager
d---s-        7/10/2015   6:11 AM                DSCResources
d---s-       11/17/2015   9:13 AM                en-US
d---s-        7/10/2015   4:04 AM                WebDownloadManager
```

The _resource module_ can contain one or more resources, and each actual _resource_ is in the DSCResources subfolder:

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d---s-        7/10/2015   6:11 AM                en-US
d---s-        7/10/2015   6:11 AM                MSFT_ArchiveResource
d---s-        7/10/2015   6:11 AM                MSFT_EnvironmentResource
d---s-        7/10/2015   6:11 AM                MSFT_GroupResource
d---s-        7/10/2015   6:11 AM                MSFT_LogResource
d---s-        7/10/2015   6:11 AM                MSFT_PackageResource
d---s-        7/10/2015   6:11 AM                MSFT_ProcessResource
d---s-        7/10/2015   6:11 AM                MSFT_RegistryResource
d---s-        7/10/2015   6:11 AM                MSFT_RoleResource
d---s-        7/10/2015   6:11 AM                MSFT_ScriptResource
d---s-        7/10/2015   6:11 AM                MSFT_ServiceResource
d---s-        7/10/2015   6:11 AM                MSFT_UserResource
d---s-        7/10/2015   6:11 AM                MSFT_WaitForAll
d---s-        7/10/2015   6:11 AM                MSFT_WaitForAny
d---s-        7/10/2015   6:11 AM                MSFT_WaitForSome
d---s-        7/10/2015   6:11 AM                MSFT_WindowsOptionalFeature
```

Those are not the actual _resource names_ that you'd use inside a configuration script, but they're obviously similar. Here's what's in MSFT_GroupResource:

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d---s-        7/10/2015   6:11 AM                en-US
-a---l        7/10/2015   4:01 AM          57990 MSFT_GroupResource.psm1
-a---l        7/10/2015   4:01 AM            834 MSFT_GroupResource.schema.mof
```

The .psm1 file is the actual code that makes the resource work, while the .schema.mof file tells DSC how to use the resource. The en-US folder...

```
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a---l        7/10/2015   6:11 AM           2080 MSFT_GroupResource.schema.mfl
-a---l        7/10/2015   6:11 AM           4850 MSFT_GroupResource.strings.psd1
```

Er, doesn't contain an XML help file. It looks like it contains English versions of whatever text strings the resource uses. In theory, other folders could provide translations. So, no help there. Only... there _is_.

```
PS C:\Windows\System32\WindowsPowerShell\v1.0\Modules\PSDesiredStateConfiguration\DSCResources\MSFT_GroupResource> Get-Content .\MSFT_GroupResource.schema.mof

[ClassVersion("1.0.0"), FriendlyName("Group")]
class MSFT_GroupResource : OMI_BaseResource
{
  [Key] string GroupName;
  [write,ValueMap{"Present", "Absent"},Values{"Present", "Absent"}] string Ensure;
  [write] string Description;
  [write] string Members[];
  [write] string MembersToInclude[];
  [write] string MembersToExclude[];
  [write,EmbeddedInstance("MSFT_Credential")] string Credential;
};
```

That's what's in the .schema.mof file. As I wrote, this tells DSC how to _use_ the resource, and that includes listing the properties that the resource requires. The resource's name - it's "friendly name," which you'd use in a configuration script - is "Group." It accepts several properties, including GroupName, Ensure, Description, Members, and so on. The Ensure property accepts "Present" or "Absent" as legal values. The GroupName property is the _key_, meaning each instance of this resource that you use within a configuration script must have a unique GroupName.

WAIT! Does that mean _you can only reference a given user group one time per configuration?_ Yup. That restriction prevents two different settings within the configuration from "fighting" with each other. This is the thing I mentioned being troublesome for partial configurations, because you could easily send two partials that each attempted to configure the same group. Once the target node LCM merged those partials, it'd realize there was a duplicate key, and fail the entire configuration.

But actually... there's no need to dig into the schema MOF. I mean, feel free, and it's always good to know what's going on under the hood, but PowerShell will parse this for you a bit.

```
PS C:\> Get-DscResource Group | Select -ExpandProperty properties

Name                 PropertyType   IsMandatory Values
----                 ------------   ----------- ------
GroupName            [string]              True {}
Credential           [PSCredential]       False {}
DependsOn            [string[]]           False {}
Description          [string]             False {}
Ensure               [string]             False {Absent, Present}
Members              [string[]]           False {}
MembersToExclude     [string[]]           False {}
MembersToInclude     [string[]]           False {}
PsDscRunAsCredential [PSCredential]       False {}
```

See? Same info, only here you also see the "magic" properties that PowerShell adds, like PsDscRunAsCredential. The Group resource predates the existence of that, which is why the resource provides its own Credential property.

OK - now you know what properties exist, and maybe some of the values they take. But what about the rest? What about Members? What format should that be in?

Well... er... the thing is, Microsoft is really developing resources at a fast clip. They're very "agile" these days, which translates to "lots of code, not so much documentation." In Microsoft's _specific_ case, they're trying hard, though. Browsing their GitHub repo at https://github.com/PowerShell/PowerShell-Docs/blob/staging/dsc/groupResource.md, I found some documentation and examples. And so... you kinda hope that, for whatever resource you're using, whoever wrote it has done something similar.

Look, let's just be honest about something for a minute. We're on the bleeding edge of computing with this DSC stuff. Things are in constant flux, and in a continuous state of improvement. If you're going to play in this space, you're going to have to accept that it's a bit more like being an open source coder, and a lot less like being a Wizard-clicking administrator. Expect to hunt for docs in GitHub. Expect to "figure things out" on your own, or by asking for help on community sites. If you're cool with that, you'll have a lot of fun with DSC. If you're not so cool with that... well... it's kinda gonna suck a little. We're in the hack-y side of Microsoft now, the side that's embraced open source and agile development. This isn't like Windows 95, where everything you needed came in a box or floated down through Windows Update. Hopefully, you're excited to be in the brave new world of Microsoft.