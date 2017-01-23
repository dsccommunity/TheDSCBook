# Introduction

Desired State Configuration, or DSC, was introduced as part of Windows Management Framework (WMF) 4.0 - the same package that included Windows PowerShell 4.0. DSC is essentially the final step originally envisioned for Windows PowerShell in inventor Jeffrey Snover's _[Monad Manifesto](https://leanpub.com/themonadmanifestoannotated)_. DSC builds upon a decade-long investment in Windows PowerShell that included broad cmdlet coverage, a robust (yet simple) scripting language, Remoting, and more. 

## What is DSC?
Many organizations - maybe even the one you work for - have written configuration documents that describe how certain computers should be configured. "In our environment, a domain controller has _this_ set of services installed, _these_ pieces of software, and is configured _this_ way." When someone sets up a new machine, they often consult these standards, which are designed to help configurations stay consistent, reliable, and secure.

At a high level, DSC simply asks you to write those configuration documents in a specific way that a computer, as well as a human, can understand. Rather than simply writing, "a domain controller should have Active Directory, DNS, and DHCP installed," you might write it like this:

```
WindowsFeature DnsServer {
	Name = "DNS"
	Ensure = "Present"
}
WindowsFeature DhcpServer {
	Name = "DHCP"
	Ensure = "Present"
}
WindowsFeature AD {
	Name = "ADDS"
	Ensure = "Present"
}
```

You would then deploy that configuration document directly to each domain controller, including any new ones you want to create. DSC "reads" that document, and does whatever is necessary to "make it happen." It checks to make sure the specific services are installed, and if they're not, it installs them. Every so often, it can re-check to make sure everything is still configured correctly, and it can even report back to you and let you know that everything is in order (or not).

The actual process is actually a little more complex (but only a little bit), and your configurations can obviously be a _lot_ more detailed and comprehensive. But, at a high level, this is what DSC does. It takes a human-readable document that is also machine-readable, and it does the work of configuring systems to match your specification.

## Not Just for Provisioning
It's very easy to fall into the trap of thinking of DSC as a way of setting up computers. "I'll push out this configuration to get the computer going, and then from there I'll manage it like I usually do." That thinking misses the entire point of DSC. 

DSC is about putting a computer into a _desired state_ of configuration, and then _keeping_ the computer in that desired state. But what you "desire" of a computer can certainly change over time. The ultimate idea behind DSC is that you _never again configure a computer manually_. Instead, when you change your mind about what a computer should look like, you modify its DSC configuration. DSC reads that new configuration, realizes that the computer doesn't look like that, and "corrects" the configuration, bringing the computer in compliance with your _new_ desired state.

## How Does DSC Compare to Group Policy?
On the surface, DSC and Group Policy seem to serve the same high-level purpose. Both of them enable you to describe what you want a computer to look like, and both of them work to keep the computer looking like that. But once you dig a little deeper, the two technologies are grossly different.

Group Policy is part and parcel of Active Directory (AD), whereas DSC has no dependency on, or real connection to, AD. 

Group Policy makes it easy to target a computer dynamically based on its domain, its location (organizational unit, or OU) within the domain, its physical site location, and more. Group Policy can further customize its application by using Windows Management Instrumentation (WMI) filters and other techniques. DSC, on the other hand, is very static. You decide ahead of time what you want a computer to look like, and there's very little application-time logic or decision-making.

Group Policy predominantly targets the Windows Registry, although Group Policy Preferences (GPP) enables additional configuration elements. Extending Group Policy to cover other things is fairly complex, requires native C++ programming, and involves significant deployment steps. DSC, on the other hand, can cover any configuration element that you can get to using .NET or Windows PowerShell. Extending DSC's capabilities is a simple matter of writing a PowerShell script, and deploying those extensions is taken care of by DSC's infrastructure.

For these reasons, DSC has primarily been focused on server workloads, since server configurations tend to be fairly unchanging. DSC is also suitable for statically configured client computers, such as computers in a call center, a student lab environment, a kiosk scenario, and so on. But because DSC doesn't readily support a lot of application-time logic, it's less suitable for the dynamic, people-centric world of client computers.

It's entirely possible that, in the future, Microsoft could rip out the guts of Group Policy and replace it with DSC. Client-side logic could still pull DSC configurations dynamically from an AD domain controller, just as Group Policy does today. It'd be a way of massively extending what Group Policy can do, while keeping all the dynamic-targeting advantages of Group Policy. But today, you'll mostly see DSC discussed in terms of server workloads and more-or-less static configurations.

## Cattle, Not Pets
DSC is one of those technologies that can be hard to grasp until you truly embrace a couple of its core design principles. It's not one of those things where you can kind of pick and choose the things you like about it; if you're going to be successful with DSC, you kind of have to "drink the Kool-Aid" and go all-in with its approach.

The first principle that you have to embrace is that _DSC is meant for scale_. If you're managing five servers, DSC is going to be irritating. DSC is useful when you need to manage a hundred servers... or a thousand... or tens of thousands. And one thing you have to accept about managing that many servers is that you have to avoid servers that are unique, special unicorns - or as Jeffrey Snover and I like to call them, _pets_. 

A pet has a name. It has a unique personality. It's special. You cuddle it, and you have a lot of fun stories about it and its history. DSC isn't so hot at managing those kinds of machines. Oh, you can do it, for sure - but you're going to find yourself running up against seeming shortcomings in DSC. Thing is, they're not shortcomings - DSC just wasn't designed with special snowflakes in mind.

DSC was designed to manage _at scale_. You can't have a thousand pets in your home, but you can have a thousand cattle on a farm, right? DSC is designed for cattle management. Cattle don't have names (well, they might have names for themselves, but nobody cares). Cattle aren't unique. They're not individually special. If one does become special - perhaps by needing customized living arrangements and a 401(k) plan - you're likely just to shoot it, eat it, and go back to managing the herd.

Adopting this approach means you will probably have to make some significant changes to how you think about server management. For example, today, you probably have unique names for all of your servers, and you probably know those names. Those are pets. If you decide to use DSC to manage a server's complete configuration, including its unique, special name, then DSC isn't going to be as efficient as it could be. Instead, you could switch to a "cattle" approach. Sure, your servers will still have unique names - but Windows makes a name up for itself when you install it. Just because the server thinks it has a name doesn't mean you need to know it, or that you need to care about it. So DSC simply ignores the server's name. It's just a cow, right? Instead, DSC assigns the server to a load balancer, or to a CNAME record in DNS, or some other abstraction. So you'll be able to refer to the server - and you might simply refer to a _group_ of them, as with a group of web servers that are all serving the same function. You just won't use a special, unique name for that server. 

This is a _huge_ shift in thinking for most Windows administrators. Fortunately, you don't need to go all-in with that concept right off the bat. DSC will still let you set things up more or less the way you do today, although you may find some parts of it seem inefficient or cumbersome. As you start to shift your thinking, though, DSC will become an easier and more efficient way to manage your machines.

## Technology, not Tool
The other thing to know about DSC is that it's a technology - not a tool. In the same way, Active Directory is a technology, and something like AD Users & Computers is a tool that you use to manage the technology. Today, Microsoft hasn't built a lot of tools to make managing DSC easier - it's a pretty manual, low-level process. That's changing a bit - I have a chapter coming up on DSC in Azure, for example, that shows some of what Microsoft has been building in the way of tools. Just understand that, right now, there are some rough edges around DSC because all we have is the technology, not any tooling to help us use the technology more effectively. But don't let that stop you.
