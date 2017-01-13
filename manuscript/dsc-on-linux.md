# DSC on Linux
Microsoft offers a free DSC client - that is, a Local Configuration Manager, or LCM - for Linux. It's important to know that this is _not_ the work of the PowerShell team, but rather of Microsoft's excellent Unix Tools team. [Here's a great getting started guide](https://msdn.microsoft.com/en-us/powershell/dsc/lnxgettingstarted), and you can access the [open source code on GitHub](https://github.com/Microsoft/PowerShell-DSC-for-Linux).

Authoring configurations is just the same, although the available resources are obviously different. Bundled in the LCM are the following resources:

* nxArchive
* nxAvailableUpdates
* nxComputer
* nxDNSServerAddress
* nxEnvironment
* nxFile
* nxFileInventory
* nxFileLine
* nxFirewall
* nxGroup
* nxIPAddress
* nxLog
* nxMySqlDatabase
* nxMySqlGrant
* nxMySqlUser
* nxOMSAgent
* nxOMSCustomLog
* nxOMSKeyMgmt
* nxOMSPlugin
* nxOMSSyslog
* nxPackage
* nxScript
* nxService
* nxSshAuthorizedKeys
* nxUser

Most are written in C. It's worth noting that DSC on Azure - that is, Azure Automation - also supports Linux by means of this LCM client. Writing your own modules for the Linux LCM is a bit more complex, as you can't necessarily just write a PowerShell script (PowerShell didn't exist on Linux when this LCM was written, so they assumed you'd be coding in C). 

The team also provides MOFs for their resources that can be installed on Windows, so you can author (getting IntelliSense and whatnot in the ISE) Linux-targeted configurations on Windows machines.