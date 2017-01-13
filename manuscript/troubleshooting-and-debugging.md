# Troubleshooting and Debugging

## Getting Eyes-On
The key thing with troubleshooting and debugging is simply _figuring out what's happening_. There are a number of ways in which you can do that.

### Interactive Push
Get a copy of a node's MOF, switch the node's LCM to Push mode, and push the MOF using **Start-DscConfiguration -Wait -Verbose** (adding in the parameter to specify the MOF, of course). You'll get some of the best diagnostic information that way, if you can take this temporary troubleshooting step.

### Get the Failing Node's Configuration
Use **Get-DscConfigurationStatus** to query the node's LCM directly. You'll get a list of failed resources, anything requesting a reboot, and a lot more. The LCM can also return historical information, as opposed to including only the most recent consistency check, when you add the **-All** parameter.

The command will return an object, one good trick is to run:

```
Get-DscConfigurationStatus | Select -Expand ResourcesNotInDesiredState
```

You'd run this right on the node, and you'll get a dump of every resource that's reporting non-compliance, along with error information.

### Check the Logs
The LCM can also report extensive information to the Debug, Operational, and Analytic event logs on the node. For example:

```
Get-WinEvent -LogName 'Microsoft-Windows-Dsc/Operational'
```

Substitute "Analytic" or "Debug" for "Operational" to see those logs; the Debug log tends to hold the most detailed information. You can also access these using the Event Viewer GUI tool, although you'll need to right-click to make the Debug and Analytic logs visible.

* The Operational log contains all outright error messages.
* The Analytic log goes into more detail, and can include any verbose messages output by resources or the LCM.
* The Debug log contains the greatest level of detail.

Each log includes a Job ID, which corresponds to a single DSC operation. Once you query the Operational log, for example, you can start to see the Job ID from the most recent consistency check. You can then limit further queries to just that job ID, if you want. You'll find some helpful commands in the xDscDiagnostics module (which is available in PowerShell Gallery, so use Install-Module to get it). One command is **Get-xDscOperation**, which includes a **-Newest** parameter that can grab just the most recent number of operations. Another command is **Trace-xDscOperation**, which can consolidate events for a given operation from all three logs.

### Stop Resource Module Caching
Normally, the LCM cashes DSC resource modules for better performance. But when you're troubleshooting or debugging a module, you may want to force resources to be re-imported each time the LCM runs. The LCM's **DebugMode** property can be set to $True, which forces the LCM to always reload DSC resources, ensuring a "fresh" run. Set this back to $False when you're done debugging, so that the engine can go back to caching resources for better performance. So, to be clear, changing this requires that you modify the LCM configuration by pushing in a new meta-MOF.

### Check the Report Server
If you've configured a node to use a report server, it will forward any reports - often more useful than what appears on-screen on the node - to that report server. It's worth querying the server to see what's  up.

## Resource Debugging
This'll become a full chapter at some point, but keep in mind the **Enable-DscDebug** and **Disable-DscDebug** commands. These enable debugging of DSC resource modules, which enables you to force resources to "break" into debug mode when the consistency check runs them. This is a neat way to "step through" the resource module's code and see what's happening. You'll usually add the **-BreakAll** parameter when enabling; see the command's documentation for more information.

## Stopping a Hung LCM
Normally, running **Stop-DscConfiguration -Force** (and of course adding the right computer name) will get a computer's LCM to stop processing. But the LCM, today, is a single-threaded process, and if it's gone off the deep end - usually the result of a resource module going off and doing some long-running task, improperly expecting interactive input, or something else, then the LCM might not respond to your stop request. In that case, try this _on the offending node_:

```
Remove-Item $env:systemRoot/system32/configuration/pending.mof -Force
Get-Process *wmi* | Stop-Process -Force
Restart-Service winrm -Force 
```

This will remove the MOF that the LCM _would have processed on its next run_; you may also need to delete current.mof and backup.mof from the same location. You can also try running **Remove-DscConfiguration**, which does essentially the same thing.

The second bit - stopping the WMI service and restarting Windows Remote Management (WinRM) - can be useful, especially on older systems. Everything in DSC actually runs out of Windows Management Instrumentation (WMI), and in cases where it's the underlying WMI service holding things up, killing the process can get the LCM to unstick. You _may_ need to manually restart the WMI service after doing this (or reboot the node). Similarly, because DSC uses WinRM to communicate, sometimes it's those communications which hang things up. Restarting WinRM (or killing the process and then starting the service) can unstick things. Thanks to Peter Burkholder for suggesting this section!






