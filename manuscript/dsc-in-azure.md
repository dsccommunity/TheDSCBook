# DSC in Azure
In the Introduction, and in the previous chapter, I've made a big deal out of the fact that DSC is a _technology_, and that using it can require a lot of manual effort because, right now, we don't have a lot of _tools_ built atop the technology. But lurking inside Azure Automation is a preview of how those tools might look. 

By the way, this isn't a sales pitch for Azure; it's a sales pitch for _tools_, and how they can make using a complex technology a lot easier.

In essence, what Azure Automation provides (for the purposes of DSC) is a Pull Server in the cloud. It isn't the standard Pull Server that you can install on-prem; with access to the Pull Server source code, the Azure team has been able to customize their offering quite a bit. But, at the end of the day, it's a Pull Server. You can copy your configuration MOFs to it, and you can drop you ZIPped-up DSC resource modules on it. Your nodes - whether they're running Azure, AWS, on-prem, or someplace else - can be pointed to this Pull Server. They provide some nice authentication options to make sure only _your_ machines are pulling configurations, too.

You can also _author_ your configurations right in the cloud, run them, and generate MOFs right there. And Azure's Pull Server supports the Reporting Server function - only they've enhanced it _with actual reports and dashboards_, so you can use an Azure-based console to track the compliance of your nodes (in-cloud and on-prem), generate reports, and so on.

Azure's Pull Server runs only on HTTPS, which means your nodes can be assured that they're connecting to _your_ pull server, and not some imposter.

Azure provides a lot of integration points, especially for Azure-based VMs, to make node enrollment easier. For example, the Azure Automation DSC console lets you configure an Azure-based VM's LCM from a graphical interface, enabling you to get machines pointed to the Pull Server more easily. That configuration can also be "injected" into new VMs, enrolling them automatically upon creation. Nodes are onboarded securely, which means Azure equips them with a client certificate. This helps ensure that only _your_ nodes can request configurations from your Pull Server.

Azure also provides tooling for some of the more complex and difficult aspects of DSC, such as managing credentials in MOFs. And keep in mind, you can use Azure DSC _even if you don't have any VMs in Azure_. It's just a fancy Pull Server, although you do have different onboarding steps for nodes that aren't running in Azure VMs.

