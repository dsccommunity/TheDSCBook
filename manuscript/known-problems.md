# Known Problems
Here, I document some of the known problems folks have run across, along with what solutions I know of.

## Error configuring the LCM: "Specified Property does not exist," "MI RESULT 12"
This seems to occur on nodes that were upgraded from WMF v4, or from a pre-release of v5. A fix was documented at <https://msdn.microsoft.com/en-us/powershell/wmf/limitation_dsc>.

## Registration Key Problems
When using a node with ConfigurationNames - and therefore, a registration key GUID, you may see failures to register, and log errors on the pull server. That was a known problem with an older version of the xPSDesiredStateConfiguration module (prior to 3.10), especially on Server Core (where you could also see errors related to a missing JET database engine). 

## maxEnvelopeSize Errors
DSC uses WS-MAN to send data between computers - such as when deploying a MOF in push mode to a node. WS-MAN has some configuration limits (exposed in the node's wsman:/ drive via PowerShell), including maxEnvelopeSize. As a rule, you should have this property set to roughly double the size of the MOF you plan to deploy.

## Reporting Server and Large Configurations
The Reporting Server provided by Microsoft has an (undocumented) upper limit on how big of an HTTP request it can receive from nodes. With especially large configurations, it's possible to exceed this (currently unknown) limit. The node will log:

```
Job {%JobID%} :
Http Client {%AgentID%} failed for WebReportManager for configuration The attempt to send status report to the server https://johntestx02/PSDSCPullServer.svc/Nodes(AgentId={%AgentID%}')/SendReport returned unexpected response code RequestEntityTooLarge..
```

This has been logged as a bug with the team, at https://windowsserver.uservoice.com/forums/301869-powershell/suggestions/15563172-node-with-large-configurations-fails-to-send-statu 

## Class-Based Resources Can't be ExclusiveResources in Partials
This is a known problem and is fixed in WMF5.1 onward.