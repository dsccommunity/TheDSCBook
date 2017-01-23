# Self-Modifying Configurations
This whole chapter grew out of a "Stupid DSC Tricks" session I did at a couple of conferences, including PowerShell + DevOps Global Summit 2016 <http://powershellsummit.org>. The idea was to try and push the boundary on DSC a little bit. One goal in particular was to create a configuration that could download pre-requisites for the actual, intended configuration.

Here's the scenario: you want to push a configuration down to a node. Fine. But perhaps the configuration will contain credentials, which you'll be encrypting of course, and so you _first_ need to get the certificate installed locally. Or perhaps you need to pre-stage some pre-requisite modules or resources, and can't use a pull server for whatever reason. Problem is, if the LCM sees that you're using resources it doesn't have, it won't even run the configuration. So you want to run a small, short, "bootstrap" configuration first, to get everything all set up, and _then_ run the "real" configuration. 

Now, look: I'm not positioning all of this as a Good and Just Idea. This chapter is more about pushing the edges of DSC to see what we can do. Think of it as a thought experiment, of sorts. But over the course of thinking through this, I've come up with what I think are some clever ideas. So this chapter will be less a how-to, and more a freewheeling stream-of-consciousness. Oh, you'll see some actual tech, too, but it's possible that the tangential ideas will end up being the most useful.

## Understanding the LCM's Processing
The LCM actually stores its configuration on disk, in /System32/Configurations. It's easy to get to. There are no registry calls or anything else - the MOF file is just a plain text file in that folder. If the LCM has run before, then there can actually be multiple files in that location:

* Pending.mof is the newest configuration the LCM has received, and is the one the LCM will use on its next consistency check.
* Current.mof is the one the LCM is running right now.
* Previous.mof is the one the LCM used on its last consistency check.
* Backup.mof is usually the same as Current.mof, and unless a new configuration was JUST received, will also be the same as Pending.mof.

Therefore, the LCM gets a configuration and saves it as Pending.mof. When the next consistency check runs, this is copied to Current.mof and Backup.mof, and the consistency check starts. When the check finishes, Current.mof is copied to Previous.mof. If there's a failure of some kind, Pending.mof won't exist, thus preventing the "failed" configuration from being picked up again.

So we can "inject" a configuration by simply writing a MOF to Pending.mof and waiting for the LCM to run its next consistency check.

Additionally, Metaconfig.mof can contain the LCM's meta-configuration. Replacing this file is a way to "inject" a new meta-configuration, thereby reconfiguring the LCM. Note that Metaconfig.mof file can only contain the **MSFT_DSCMetaConfiguration** and the **MSFT_KeyValuePair** sections. But MOF files produced by running a PowerShell configuration script may contain an **OMI_ConfigurationDocument** section, which must be removed.

## The Basic Self-Modifying Workflow
So, the basic idea is to push (or make available on a pull server) a "bootstrap" configuration. This is intended only to get whatever pre-requisite stuff you need in place. It looks and works just like a normal configuration... except, at the end, it does something a little different. It "injects" the _real_ configuration, and possibly "injects" a new LCM meta-configuration.

So, with the understanding that this is all a little theoretical, here's a model of the idea:

```
Configuration Bootstrap {

	Node whatever {
	
		Prerequisite One {
			Property = Value
			Property = Value
		}
		
		Prerequisite Two {
			Property = Value
			Property = Value
		}
	
		File [string] #ResourceName
		{
    		DestinationPath = pending.mof
    		Ensure = Present 
    		SourcePath = \\server\folder\source.mof
    		DependsOn = [Prerequisite]One,[Prerequisite]Two
		}

	}

}
```

So in broad strokes, the idea is to have one or more resources - "Prerequisite" resources, in this example - set up whatever you need on the node. A final resource injects the "real" MOF. In this case, I'm using the File resource to copy the MOF from a file server, so the presumption is that the MOF has already been created and staged there.

## Options
So, the basic theme is to create a "bootstrap" configuration, which will set up any prerequisites, and then somehow engage the "real" configuration. You've got a lot of options for getting there.

1. Set up a pull server where the bootstrap configurations reside. Configure the node LCM to pull from that server. The bootstrap then modifies the LCM to pull from a different pull server, where the "real" configuration resides.
2. Push the bootstrap to the node, and the bootstrap configures the LCM to pull the "real" configuration from a pull server.
3. Have the node pull the bootstrap configuration by GUID. The bootstrap then reconfigures the LCM to use configuration names, allowing it to pull the "real" configuration from the same pull server.
4. Push the bootstrap to the node, and have the bootstrap directly inject a new, "real" configuration. This is basically what my above example does.
5. Push the bootstrap to the node, or have the node pull it. This does whatever it needs to do, and then runs a PowerShell configuration script, producing a MOF. This MOF is the "real" configuration, and could be written locally (i.e. injected), or written to a pull server. 

All of this "reconfigure the LCM" stuff can be done by having the bootstrap write a new meta-configuration MOF directly on the node. The LCM will pick up the "injected" meta-configuration MOF pretty quickly, thereby reconfiguring the LCM.

My fifth suggestion in the list above offers some interesting possibilities. Because, in that situation, the configuration _script_ is running _on the target node_, the script can include logic that dynamically adjusts the final MOF based on local conditions. This means that a single complex configuration script could, when run on different nodes, produce entirely different MOFs. And as I noted in the list, the MOF could be written right back to a pull server, overwriting the "bootstrap" configuration and allowing the node to pull the new, dynamically produced MOF.

The possibilities really are endless, which is why I've found this to be such an interesting line of experimentation.

## A Problem to Consider
One problem with "injecting" a MOF into the local node is the encryption introduced in v5. The deal is that the various MOF files stored by the LCM - including pending.mof, which is the MOF it'll process on its next consistency check - are encrypted on disk. Both Start-DscConfiguration and Publish-DscConfiguration know how to encrypt the file correctly (technically, the encryption is done _on_ the target node, not before the file is sent to the target node), but I'm not currently clean on how you might manually encrypt the MOF. If pending.mof isn't encrypted correctly, then the LCM won't read it, so your "injected" configuration won't work.

I've toyed with writing the MOF locally, but not in pending.mof, and then using Publish-DscConfiguration to send the MOF to the local host. So, the bootstrap configuration would create a local MOF (copy it from someplace else, create it locally, or whatever), and then basically "publish" the MOF to itself. I've run into some credential problems with this, though. The bootstrap is being run by the LCM, which runs as SYSTEM, which doesn't have permission to remote to itself. My only solution has been to provide a credential for the publish operation - but that necessitates encrypting the credential in the MOF. In order for that to work, you have to have the decryption certificate already staged - and getting the certificate there is one of the things I'd want to use the bootstrap for. So it's a bit of a Catch-22. I'm still playing with it, and I believe it should be possible to encrypt the MOF directly on the node, without having to "publish" it.

## Crazy Ideas for What the Bootstrap Can Do
So what sorts of prerequisites might a bootstrap configuration handle?

* Install certificates for decrypting credentials in a MOF
* Install DSC resources that can't be delivered from a pull server
* Install modules that can't be deployed in any other way

And in addition to deploying prerequisites, the ability - as I outlined already - to have a PowerShell configuration script run _on the node_ opens up a world of interesting possibilities for producing more "locally aware" MOFs that adjust themselves based on local conditions.

Anyway... it's all an ongoing experiment, which is why I included this so near the end of the book, but I think it's a useful way of thinking about how DSC can be twisted to solve different needs. And certainly, it's a demonstration of how important it is to understand how DSC works under the hood!