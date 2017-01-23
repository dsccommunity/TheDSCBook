# Infrastructure Prerequisites
There are a few things that we assume you have in place in order for DSC to work. It's not a huge list, but it's an important one.

* All nodes that will be used to author, push, or receive configurations are running PowerShell v5 or later. Keep in mind, this book doesn't cover v4.
* WS-Management (WS-MAN) traffic is permitted on the network. This isn't PowerShell Remoting ; DSC uses CIM, which also runs over the WS-MAN protocol. It will be enabled by default on nodes that have PowerShell v5 installed, but you must ensure it's not being blocked by firewalls or other network elements.
* For authoring nodes, we assume that they have Internet connectivity to download new DSC-related resources, and that they have the PowerShell ISE installed (which would be the case by default on Windows client computers).

