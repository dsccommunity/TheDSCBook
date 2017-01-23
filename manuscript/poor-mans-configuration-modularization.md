# Poor Man's Configuration Modularization
Unless you're in the very smallest of environments, or only using DSC for a very small subset of nodes, you're going to end up with more than one configuration script. Most of the time, you'll probably use some gut-instinct measurement of the configuration's difference, or "delta," from other configurations to decide when to make a new one.

For example, suppose you have two kinds of application web server. For each kind, you have a half-dozen actual machines. The two are very similar: they have the same firewall settings, basic IIS install, and so on. The main difference is perhaps some application pool settings, and the source files they use. So let's say they're about 90% the same, with a 10% delta. You might decide, "hey, 10% ain't so much, I'll just make one configuration, and use some Configuration Data filtering logic stuff to handle that 10%." 

The percentage of delta that falls within your "comfort level" for having one configuration, versus two different ones will be very much a personal thing. But at some point, the "delta" between the two configurations will be large enough that you'll want to make two separate configuration scripts - while still modularizing, as much as possible, the common elements of them.

This chapter will present the easiest approach, at least conceptually. The next two chapters, which cover composite and partial configurations, will present different approaches. All of these have their own pros and cons, and you'll simply have to decide which works best for _your_ organization.

## Dot Sourcing
Suppose you have a configuration that's something like the following. Notice that I'm not using real DSC resources here - this isn't meant to be an exhaustive syntactic example, but rather a structural one.

```
configuration MyConfig {

  Node $AllNodes.Where({ $_.ServerType -eq 'AppServer' }) {

    WindowsFeature IIS {
      Name = 'Web-Server'
      Ensure = 'Present'
    }
    
    SourceFiles AppFiles {
      Source = '\\master\distrbution\appserver'
      Destination = 'c:\inetpub\approot'
      SyncMode = 'checksum'
    }

  }

  Node $AllNodes.Where({ $_.ServerType -eq 'SalesServer' }) {

    WindowsFeature IIS {
      Name = 'Web-Server'
      Ensure = 'Present'
    }
    
    SourceFiles AppFiles {
      Source = '\\master\distrbution\salesserver'
      Destination = 'c:\inetpub\salesroot'
      SyncMode = 'checksum'
    }

  }

}
```

So I'm configuring two kinds of nodes. They both have a common element - the IIS bit - and an element that differs. Because this is such a simple example, the delta between the two is around 50% - meaning half the configuration is different. So I could modularize this a bit.

I'll start by creating a file called ./IISStuff.ps1. I'm going to put it in the same folder as the actual configuration script, and it will look like this:

```
    WindowsFeature IIS {
      Name = 'Web-Server'
      Ensure = 'Present'
    }
```

That's not a valid configuration script! The ISE won't even color-code it or error-check it correctly. It's just a _fragment_. Now, in my main file, which I'll call .\AppServers.ps1, I'll do this:

```
configuration MyConfig {

  Node $AllNodes.Where({ $_.ServerType -eq 'AppServer' }) {

    . .\IISStuff.ps1
        
    SourceFiles AppFiles {
      Source = '\\master\distrbution\appserver'
      Destination = 'c:\inetpub\approot'
      SyncMode = 'checksum'
    }

  }

  Node $AllNodes.Where({ $_.ServerType -eq 'SalesServer' }) {

   . .\IISStuff.ps1
    
    SourceFiles AppFiles {
      Source = '\\master\distrbution\salesserver'
      Destination = 'c:\inetpub\salesroot'
      SyncMode = 'checksum'
    }

  }

}
```

I've taken a first step at "modularizing" the common code. I've moved it into an external file, which is then dot-sourced into the configurations. Now from here, you could take the approach much further. You could have an entirely different configuration script file for each node type, each using IISStuff.ps1 as needed. You could obviously dot-source _multiple_ files containing commonly used fragments. 

## Approach Analysis
I'm presenting this approach _not_ because I see it used very often or because I think it's a great idea. I'm presenting it _mainly_ because it's conceptually easy for most PowerShell scripters to understand. But we need to do some analysis of its pros and cons.

* PRO: Unlike a Composite configuration, dot-sourcing doesn't require you to deploy anything to your nodes. 
* PRO: Unlike a Partial configuration, you'll be producing a single MOF-per-node that's complete and ready to deploy; you'll reduce errors like duplicate keys.
* CON: You can't ever look at the entire configuration "in one place, in front of your face," because it's spread across multiple files. That's also true with Partial configurations and, to a point, with Composite configurations.
* CON: Dot-sourcing just feels a little... messy. It's not as structured as approaches like Composite configurations or, to a point, Partial configurations.
* CON: You can't test the individual fragments in a standalone fashion, so you're not _truly_ modularizing, here. That's not the case with either Composite or Partial configurations, which _can_ be tested independently. Sort of. You certainly _could_ work up "test configurations" that dot-sourced and tested individual fragments, but it can get messy.

But with this dot-sourcing "modularization" in mind, we can start exploring the more "official" ways of modularizing DSC configurations.