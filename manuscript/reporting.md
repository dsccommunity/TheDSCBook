# Reporting
Sitting here writing this in July 2016, I have to make sure you understand that the default Report Server ain't much to look at. In fact, at PowerShell + DevOps Global Summit 2016, PowerShell team leads made it clear that the Report Server _was only ever intended as a sample_ of what you could do. Azure, as I've already pointed out, takes reporting a lot further, and is a strong reason to use Azure Automation as your pull server. Maybe all that'll change someday, and maybe not. For now, my goal is to help you understand what the default server is and does. Note that this chapter only applies to WMF v5; the "Compliance Server" in v4 doesn't work the same way, and has been discontinued.

## Understanding the Default Report Server
If configured to use a reporting server, each node's LCM will send a status packet to that reporting server. This happens each time the node runs a consistency check on its configuration, which means it can't happen if the LCM's application mode is disabled. The status information is sent in a specific format, and the Report Server's job is simply to accept it and store it in a database.

The default Report Server uses the Extensible Storage Engine, or ESE - the same database that Active Directory uses. _Presently_, the database schema is unknown, and it's not practical to move the data storage elsewhere. That's a potential weakness, as a lot of us would like to relocate that information to, say, a SQL Server that can be made more available and scalable. We're hopeful, as a community, that Microsoft can one day release the source code of their "sample" Report Server, so we can see about building our own.

## Querying Report Data
The default Report Server lets you query node data by making calls to the same Web service that nodes report to. It's a REST call, which means you can simply use Invoke-WebRequest:

```
$requestUri = "https://my-report-server/PSDSCReportServer.svc/Nodes(AgentId='12345')/Reports"
$request = Invoke-WebRequest -Uri $requestUri  -ContentType "application/json;odata=minimalmetadata;streaming=true;charset=utf-8" `
               -UseBasicParsing -Headers @{Accept = "application/json";ProtocolVersion = "2.0"} `
               -ErrorAction SilentlyContinue -ErrorVariable ev
```

That AgentId is the tricky bit. I've shown it here as 12345, which it will never be. It's a GUID, and it's configured as part of the LCM. Each LCM makes up its own AgentId. The result of this - stored in $request in the above example - is a JSON object.

```
$report = $request.content | ConvertFrom-Json
$report | Format-List -Prop *
```

In this case, $report will often contain multiple objects, since the Report Server keeps multiple check-ins for each node. Each of those objects will look something like this:

```
JobId                : 019dfbe5-f99f-11e5-80c6-001dd8b8065c
OperationType        : Consistency
RefreshMode          : Pull
Status               : Success
ReportFormatVersion  : 2.0
ConfigurationVersion : 2.0.0
StartTime            : 04/03/2016 06:21:43
EndTime              : 04/03/2016 06:22:04
RebootRequested      : False
Errors               : {}
StatusData           : {{"StartDate":"2016-04-03T06:21:43.7220000-07:00","IPV6Addresses":["2001:4898:d8:f2f2:852b:b255:b071:283b","fe80::852b:b255:b071
                       :283b%12","::2000:0:0:0","::1","::2000:0:0:0"],"DurationInSeconds":"21","JobID":"{019DFBE5-F99F-11E5-80C6-001DD8B8065C}","Curren
                       tChecksum":"A7797571CB9C3AF4D74C39A0FDA11DAF33273349E1182385528FFC1E47151F7F","MetaData":"Author: configAuthor; Name: 
                       Sample_ArchiveFirewall; Version: 2.0.0; GenerationDate: 04/01/2016 15:23:30; GenerationHost: CONTOSO-PullSrv;","RebootRequested":"False
                       ","Status":"Success","IPV4Addresses":["10.240.179.151","127.0.0.1"],"LCMVersion":"2.0","ResourcesNotInDesiredState":[{"SourceInf
                       o":"C:\\ReportTest\\Sample_xFirewall_AddFirewallRule.ps1::23::9::xFirewall","ModuleName":"xNetworking","DurationInSeconds":"8.785",
                       "InstanceName":"Firewall","StartDate":"2016-04-03T06:21:56.4650000-07:00","ResourceName":"xFirewall","ModuleVersion":"2.7.0.0","
                       RebootRequested":"False","ResourceId":"[xFirewall]Firewall","ConfigurationName":"Sample_ArchiveFirewall","InDesiredState":"False
                       "}],"NumberOfResources":"2","Type":"Consistency","HostName":"CONTOSO-PULLCLI","ResourcesInDesiredState":[{"SourceInfo":"C:\\ReportTest\\Sample_xFirewall_AddFirewallRule.ps1::16::9::Archive","ModuleName":"PSDesiredStateConfiguration","DurationInSeconds":"1.848",
                       "InstanceName":"ArchiveExample","StartDate":"2016-04-03T06:21:56.4650000-07:00","ResourceName":"Archive","ModuleVersion":"1.1","
                       RebootRequested":"False","ResourceId":"[Archive]ArchiveExample","ConfigurationName":"Sample_ArchiveFirewall","InDesiredState":"T
                       rue"}],"MACAddresses":["00-1D-D8-B8-06-5C","00-00-00-00-00-00-00-E0"],"MetaConfiguration":{"AgentId":"52DA826D-00DE-4166-8ACB-73F2B46A7E00",
                       "ConfigurationDownloadManagers":[{"SourceInfo":"C:\\ReportTest\\LCMConfig.ps1::14::9::ConfigurationRepositoryWeb","A
                       llowUnsecureConnection":"True","ServerURL":"http://CONTOSO-PullSrv:8080/PSDSCPullServer.svc","RegistrationKey":"","ResourceId":"[Config
                       urationRepositoryWeb]CONTOSO-PullSrv","ConfigurationNames":["ClientConfig"]}],"ActionAfterReboot":"ContinueConfiguration","LCMCo
                       mpatibleVersions":["1.0","2.0"],"LCMState":"Idle","ResourceModuleManagers":[],"ReportManagers":[{"AllowUnsecureConnection":"True
                       ","RegistrationKey":"","ServerURL":"http://CONTOSO-PullSrv:8080/PSDSCPullServer.svc","ResourceId":"[ReportServerWeb]CONTOSO-PullSrv","S
                       ourceInfo":"C:\\ReportTest\\LCMConfig.ps1::24::9::ReportServerWeb"}],"StatusRetentionTimeInDays":"10","LCMVersion":"2.0","Config
                       urationMode":"ApplyAndMonitor","RefreshFrequencyMins":"30","RebootNodeIfNeeded":"True","RefreshMode":"Pull","DebugMode":["NONE"]
                       ,"LCMStateDetail":"","AllowModuleOverwrite":"False","ConfigurationModeFrequencyMins":"15"},"Locale":"en-US","Mode":"Pull"}}
AdditionalData       : {}
```

So you can see, there's some basic information before you get to the monster StatusData property, which includes a lot of detail about the node. It also contains a dump of all the configuration items that the LCM processed, so with a large configuration, this will be a LOT of data. The StatusData is itself a JSON object, so you could run it through ConvertFrom-Json to break that data out and make it a bit more readable. You'll also notice a HostName property buried in there, which is how you can start to map host names to AgendId GUIDs. Some things to note:

* You get a StartDate property that tells you when the consistency check ran. This allows you to find the most recent one.
* You get a DurationInSeconds property that tells you how long the check ran.
* You can tell if there's a reboot pending.
* You can see the Status, which will be Success _if the LCM finished without error_; Status does _not_ necessarily mean the LCM is within compliance.
* ResourcesNotInDesiredState tells you which configuration settings aren't compliant. In this example, the Firewall resource isn't compliant. This can't tell you _why_, but it does tell you where to start investigating. ResourcesNotInDesiredState (and the companion ResourcesInDesiredState) are also JSON objects, and you can convert them to read them more easily.

## The AgentId
The AgentId is perhaps the most vexing part of this setup. You'll notice, in the chapter on Configuring the LCM, that my LCM didn't display an AgentId. That's because the AgentId is dynamically generated the first time the LCM registers with a pull server. From then on, that AgentId is used to identify the node to the pull server and any Report Server that you've configured. 

The problem is that most of us want to know the node's host name. I've seen a few novel approaches to solving this. One was a custom DSC resource which took the local AgentID, and then stored that in a "spare" attribute of the computer object in Active Directory. A similar approach registered DNS records, enabling you to query a host name to get its AgentId, or vice-versa. A third did basically the same thing in a custom database set up for that purpose. 

I think a broader weakness just comes from the fact that the Report Server is a "sample." For example, you can't easily query it for just non-compliant nodes, or for nodes that need a reboot. You can't query for host names. Again, the Report Server included "in the box" just wasn't designed to be a robust tool. Until someone writes something better, though, it's what we've got to work with.