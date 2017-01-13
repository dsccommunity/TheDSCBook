# The Scaling Question
"How can we make DSC scale?" is a question that I've struggled to help people answer. It wasn't until an early-adopter of this book emailed me, suggesting I answer the question in this book, that I started to wrap my head around what I think is a good narrative. He asked:

> Don,
> Something I know we would love to see is how to scale DSC to hundreds of nodes. What keeps coming up for us is where/how to keep our "raw" configuration data. Something that manages node information (name, IP, domain, standardRoleA, standardRoleB, roleC, roleCparam1, roleCparam2). Put it in a database (but we think it might not be flexible enough) or json files? Is it really just Puppet or Chef? Seems like there is a space out there for what we think we're looking for - and maybe an advanced chapter with your thoughts down the road. 
> Cheers,
> Joe

This winds up being something I think I touched on at the beginning of the book, but it's an important narrative. It's also complex - so I hope you'll bear with me. Also, and this is important, it's _still evolving_. What I'm going to try and share is the long-term vision, which is steadily coming to fruition, and I hope you'll understand that the current "missing bits" won't stay missing for long.

## DSC Already Scales - But You Don't
It's important that we draw a distinction between scaling the _technology_ and scaling your _processes for managing the technology_. DSC, as a platform, already scales great. There are only two bottlenecks in the system, and they're both a Web server: the pull server and the Report server. While the Report server is a bit of an evolving story, Azure Automation shows us that you can absolutely create scale there. And the pull server is *just a Web server*, so you can scale it like any other IIS box.

What doesn't scale - and what Joe's question crystallized in my mind - is our processes for managing everything up to the point a MOF goes onto the pull server. How do we keep track of all our configurations, partials, composites, and whatnot? How do we remember which roles and fragments have been assigned to each server? How do we track the unique, per-machine information - like IP addresses - that need to be injected into Configuration Data blocks?

There are two answers.

### Answer 1: Tooling
What we need is tools. And, as Joe suggests in his question, those tools might already exist in the form of Chef, Puppet, and so on. Or, you could roll your own, creating a database and some kind of management front-end for it. That latter suggestion isn't so far-fetched, either; back in the PowerShell 1.0 days, MySpace.com essentially rolled their own configuration management system that accomplished a lot of what DSC aims to do.

Microsoft hasn't yet produced their own tooling for this, although Azure Automation - more so than System Center - is starting to take steps in that direction. But I'd argue that Answer 1 isn't the right answer. In fact... 

### Answer 2: Your Brain
...I'll argue that the question is wrong. Instead of asking ourselves how to scale our management processes, and how to create or buy tools that will enable that scale, I suggest we should be asking _how to get rid of those processes_. We're all essentially managing our servers the same way we have been since the 1990s, and we've used tools to make that more scalable. Deploying software to 10 clients is easy, but 10,000? Get SCCM! Use Group Policy! But all we're doing there is _automating_ the same processes - we haven't actually _created processes that work at the needed scale_. 

And so let's begin the narrative. You're going to spot numerous potential flaws in my story here, but stick with it, because I've had pieces of this conversation with other people already. I'm going to try and anticipate the arguments and see if I can't counter them.

## Let's Set the Stage
Imagine that we work for an organization like Joe's, where we have numerous different server roles to keep track of. You probably don't need to imagine very hard, since you probably have the same thing. You've got some infrastructure servers, running Active Directory, DNS, DHCP, and so on. You have some application servers. Some SQL Server machines. And all of these probably have some common configuration elements, like management agents, security settings, and so on.

Let's focus on those infrastructure servers as a piece of our example. Let's say you have eight domain controllers, and some number of those run DHCP Server and provide DNS services. Why eight? Well, probably because you figured out that's how many DCs you need in that location to be able to support the morning logon slam. And so you provision those machines, and you keep track of them. Now, AD actually has one scale problem that Microsoft hasn't addressed, and that's the need for a copy of the AD database to be local on the domain controller. That's actually unusual for Microsoft, because almost every other "database" they have - SQL Server, Exchange Server, you name it - can have the actual data files exist elsewhere, like on a SAN or SMB 3.0 file server. In fact, it suggests that AD is either going to evolve soon, or change significantly. But for now, it's important to keep in mind the fact that each DC needs a local database, and it can only get one by replicating with other DCs. Fine.

## Raise Cattle, Not Pets
The problem is that you think of each DC as a unique and special individual - a pet. You monitor each one. You know when each one is working, or when it isn't. You manage updates on each one. Each one has a name, and you know that name. Each one has an IP address, and you track that - probably, God help us, in an Excel spreadsheet.

Why?

Let me propose an alternative. Suppose you have a monitoring system - SCOM, or OMS, or whatever. It knows, because you've told it, that you need eight DCs online in the mornings. But perhaps it also knows that, for much of the rest of the day, you only need three or four DCs. So throughout the day, it monitors three or four running DCs (I'm assuming these are in virtual machines, because why not?). As load starts to increase, the monitoring solution spins up DC five, and six, and seven, and so on. Perhaps you even have it do that automatically, early in the morning, anticipating the logon rush. Those DCs are never offline for more than a day, and you can ensure they have plenty of time to replicate changes. 

But if one of those DCs is _running_ but not _working_, your monitoring solution doesn't try to fix it. It just kills it. It removes it from the domain, deletes the VM, and spins up a new copy. Because this isn't a _pet_, it's a _cow_. And when cows get uppity, you shoot them, eat them, and buy another cow. 

Why in the world do you care what the DC's IP address is? You shouldn't. Nobody else does. It registers itself with DNS, so you can easily find it. If you insist, you could create a reservation for it in DHCP, and then you'd always know its IP address, but why bother? You can certainly ensure it gets an IP address from a pool you've designated, so it'll have a _good_ IP address, but honestly - why do you care what it is? "Well, what if DDNS doesn't work, and the DC can't register itself?" Well, then you shouldn't have built a DDNS layer that can break down entirely. Fix your infrastructure.

And why, for that matter, do you care what the DC's name is? Windows will make up a name for the machine when the OS installs. It'll register itself as a domain controller. Clients will be able to find it - it's not like you give your users a list of DC names anyway. Maybe you have a DSC resource that ensures the randomly-chosen name is in fact unique, but otherwise - why do you care?

When your orchestration/monitoring system spins up a new DC, it can certainly add it to a "pool" of some kind, so it knows to monitor that new machine as a DC. At the end of the day, all you should care about is whether the right number of DCs exist, are running, and are responding to requests - and existing monitoring solutions can do all of that for you. Existing monitoring solutions can kick off corrective actions, too - starting up more VMs or commissioning new DCs. What value, exactly, do _you_, as a slow-moving ape descendent, hope to bring to that process? People should be making _decisions_: "We need eight of these cows at all times." Let the computers do the rest, and don't sweat the details.

## Enter Containers
With the rise of containers, this story becomes even more important. We stop loading multiple roles on a VM, and instead run everything in a container. Granted, we can't run _everything_ that way today, but this story is the reason to move in that direction. DHCP server quit responding? Screw it, kill the thing and spin up a new container - it'll take 30 seconds. "High availability" becomes something you worry about in the storage and networking layers, not in the "client interface" layer - because you just spin up new containers as needed. And with containers spinning up and down all the time, you especially don't care about unique information like IP addresses and names. Do cows have names? No. Well, maybe they think they do, but I don't care.

## Rock-Solid Infrastructure
This means your infrastructure - DHCP, DNS, and so on - needs to be solid. And those are easy to make solid, especially DNS. And frankly, DNS is _really_ the key. With IPv6 in particular, computers are more than capable of making up their own IP addresses, _and that's fine_. This obsessive need to micro-manage IP addresses is outdated and insane. Want to remove a failure point from your environment? Start using IPv6 and ditch DHCP. Let computers make up their own addresses and register them with DNS. That's how we'll find them. Give computers a DSC resource that "registers" each computer's "role" in appropriate groups or pools or whatever, so we can manage them by role. 

Yeah, I'm getting a teeny bit ahead of the state of the art, here - but you see where I'm going. We don't need to know more about the details of our IT estate. We need to know _less_, and we need to have processes that work with less detail. Let the computers keep track of the details, and we'll just paint the big picture for them.

## Getting Back to DSC
So how does this relate back to DSC? Well, I'd argue that you don't need a Configuration Management Database. A CMDB represents a static view of your environment, and your environment is not static. Instead, you need to set _policies_ in a monitoring system. "We need 12 Web servers running this application, they need to be spread between these two data centers, and they each need to conform to the following configuration document. Go." _And that's it._ 

Yes, you're going to need tooling. DSC is part of that, in that it gives you a way of creating a configuration document that defines a certain role. But _you_ don't worry about _which machines_ that role is assigned to. Your monitoring and orchestration system does. "Hey, um, we only have zero of these, and we're supposed to have 12! Go-go-Gadget-deployment!" You define _what's needed_, not how to make it happen.

"But how can I tell which machines are running that role?" Why do you care? "Because if they break I have to fix them!" No, your monitoring system has to notice they are broken, and _kill_ them. You don't fix cows, you eat cows. And then buy new cows. So DSC plays an important role in defining "what a machine should look like for x role." But you don't need a big database keeping track of all those configuration fragments and documents; you just need them in a repository or gallery that your monitoring system can access. 

Orchestration and monitoring is where our industry is going. And it isn't so that you can better automate the processes you have. It's so you can _delete_ the processes you have, and adopt new processes that require less input, less granular human interaction, and less intervention. The reason people have trouble fitting DSC into their existing processes is because _those processes are outdated_, and DSC (and technologies like it) are trying to elevate the conversation to a different level.

Yes, it's going to take a lot of work and evolution to get to this end-game vision - but this _is_ where the industry (not just Microsoft) is going. 

## The Perfect Example
And if you want a real-live working example, look no further than Amazon Web Services. I use this example a lot because it's a _very good example_, and it proves without a doubt that the vision is right. And this example _is a thing_. It's real, it exists, it's mature, and you can buy it right now. It's a model you can replicate in your own environment. This is not visionary or forward-thinking, it's physical and it's right here, right now.

Amazon already has a front-end load-balancing service. They actually have a few. And they already have a service called Elastic Beanstalk. Here's how it works: you create a simple text document, not unlike a DSC configuration, that describes what a server should look like. You define a Git repository where application code lives, pre-requisite packages the application needs, and all that. In their monitoring-and-load-balancing service - which is both a load balancer and a monitor - you lay out your desired performance characteristics for a server, like how much load a given server should be allowed to handle. _And then you go away_.

Beanstalk spins up a new VM, and configures it according to your document. It gets whatever code it needs, adds the VM to the load balancer, and starts monitoring it. Nobody knows what the machine name is, and nobody cares. Nobody knows what its IP address is, and nobody cares. Oh, sure, the _computers_ know, but it's all in their heads. No human knows. After a bit, your server starts to hit max workload. Beanstalk spins up a new one, adds it to the load balancer, and configures it. Nobody had to push a button. 

So now you need to make some updates to the environment - some pre-req package has a new version, and you've made some changes to the application code. You update your configuration document, run a command, and Beanstalk spins up _new_ VMs that conform to the _new_ configuration. _It doesn't try to fix the old VMs!!!!_ It just _kills_ them, removes them from the load balancer, and replaces them with the new, shiny VMs. Because _nobody cares about old VMs._ They're gone, and the new ones are running. 

One day, one of the VMs stops responding correctly. You've configured the monitoring criteria, and perhaps it's failing some HTTP check or whatever. So the system _kills_ it, and spins up a new one. Load balancer updated, nobody's the wiser. And nobody cares about those VMs, because there is _nothing_ unique about them that any human being knows. They're cattle.

And you can absolutely run almost every computer in your environment this way. Yes, I know you've got some odd legacy stuff that might not be able to hit this exact model, but you can certainly implement a lot of this model, even with old applications running in VMs. And you can certainly move toward this model going forward. You actually have most of the tools you need, and PowerShell can help you glue them together to achieve exactly this.

And that, ultimately, is what DSC is meant to help build.
