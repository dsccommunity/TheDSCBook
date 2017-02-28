# Security and DSC
There are a lot of security concerns with DSC. Perhaps the first that jumps to mind is, "our configurations contain a lot of important and proprietary information!" True. Also true is, "if someone can modify our configurations, then they can control our environment." So we'll try and address both of these concerns.

## A Word on Certificates
Certificates play a strong role in DSC security. If you work in an organization that doesn't yet have a Public Key Infrastructure (PKI) set up, then you need to start there. Seriously, you can't do this stuff without PKI. Period. _Do not even think about using self signed certificates_, because they are Satan's handmaidens. Self-signed certificates will get you a false sense of security, but they will not, in fact, provide you with any protection. They are evil, misleading, wrong, and probably cause cancer.

DSC uses, for the most part, three types of certificates. Now, while all certificates are technologically the same in terms of the math and whatnot, certificates have to be "marked" for specific uses. These "use markings" indicate how much diligence was put into the certificate's issuance. A certificate used for encrypting email, for example, doesn't usually require much diligence in terms of verifying someone's identity. You just send the certificate to the mailbox it's intended to secure, and assume that whoever has access to that mailbox _should_ have access. A code-signing certificate, on the other hand, requires a lot more diligence. You want the certificate indisputably to identify the developer (usually a company, not a person), so that if the code turns out to be malicious, you know who to go kill. 

DSC uses:

* SSL certificates. These require moderately high diligence _not_ because they provide encryption, but because they identify a server that is handing out configurations. You _do not_ want your nodes connecting to some random web server to obtain their configurations! The SSL certificate ensures that the node is connecting to the right machine. For this reason, you should also avoid "wildcard" certificates - you don't want nodes connecting to "just any old" server, but rather to the specific server specified in their configurations.
* Client certificates. This isn't actually used by DSC per se, but you can certainly configure your web server to require client certificate authentication. This prevents unknown nodes from trying to grab your configuration data. Again - this isn't something you set up as part of DSC; client certificates would be issued to the node (perhaps through Active Directory auto-enrollment), and IIS (or whatever Web server you're using) would be configured to require the certificate.
* Document certificates. These are used to encrypt sensitive information like passwords in a MOF file. As of WMF5 and later, this is basically just a normal email encryption certificate, but it explicitly needs to be "marked" as usable for Document Encryption. This can be tough: in Windows Server 2012 R2, for example, ADCS doesn't provide a built in template for this kind of certificate, so you'll have to make your own.

So: SSL certificates on the pull server(s). Perhaps client certificates on the nodes, if you're doing that. Definitely document certificates for encrypting MOF data. No self-signed certificates. Got it?

## Securing the Pull Server
The pull server is just a web server (well, usually; it could be a file server, which is actually a bit tougher to secure to the same level). You secure web servers by using SSL certificates. You also need to lock down your DNS infrastructure, to make sure nobody can hijack the pull server's DNS name and redirect traffic elsewhere. 

**Never run a non-HTTPS pull server, ever, anywhere, for any reason, even when you're just building a pilot or a lab**. Never allow HTTP to become an easy habit. If HTTPS isn't easy, focus on making it easy. That's the investment that'll pay off. 

You also need to lock down the actual files on the pull server. Make sure any SMB share permissions, and the NTFS permissions, are appropriate for your environment. "Everyone:Read" is not appropriate. Only designated configuration authors, and your web server, should be able to read the configurations. Only designated authors should be able to write those files.

Be smart with the pull server's security.

And some technical notes: Prior to WMF 5.1, the LCM only supported SSL 3.0 and TLS 1.0. If your server was doing anything else, the LCM would seize up. As of WMF 5.1, SSL 3.0 is no longer supported, and TLS 1.0, 1.1, and 1.2 are supported. You should definitely configure your server to use the most secure protocol available.

## Securing Credentials in Configurations
DSC will not allow you, under normal conditions, to include clear-text passwords in a MOF file. However, you're very frequently going to need to include passwords, mainly when providing alternate credentials to a resource. So how do you do it? There are two options:

1. Flip the switch that tells DSC to allow plain-text passwords. This is blindingly stupid, and I'm not even going to show you how. It's on the Internet, if you're bound and determined to do the wrong thing.
2. Encrypt the password.

Option 2 is what we'll be pursuing. This requires, as I've outlined above, a certificate - and again, from WMF5 onward, the certificate must be "marked" for the "Document Encryption" purpose. There are a bunch of ways to generate such certificates, and in ADCS in Windows Server 2012 R2 onward, you can pretty easily create a certificate template that'll do this. [There's a good overview of this encryption stuff](https://msdn.microsoft.com/en-us/powershell/dsc/securemof) online; I'm going to walk through the basics and hit on some of the fine points.

### You Need a Certificate
For right now, I'm going to assume you know how to issue certificates to your nodes. This is a section I'll expand on in a future edition of this book, though. Whatever certificate you issue will need to be installed - and this is important - in the _machine_ store of the node, not in a _user_ store. If you look at the CERT: drive in PowerShell, the machine store is easy to distinguish.

The nodes need the actual certificate installed; _wherever you are authoring configurations_ need the _public key_ from the certificate, and the certificate's thumbprint. You'll use the public key to encrypt, and the thumbprint to tell the node which certificate was used. The node will grab its copy of the certificate, which includes the _private_ key, and use that key to decrypt the data.

The certificate needs to be precisely created. Its "Key Usage" must contain 'KeyEncipherment' and 'DataEncipherment', and should not contain: 'Digital Signature'. Further, the "Enhanced Key Usage" section of the certificate must include "Document Encryption (1.3.6.1.4.1.311.80.1)", and should not contain: "Client Authentication (1.3.6.1.5.5.7.3.2)" or "Server Authentication (1.3.6.1.5.5.7.3.1)". The Provider for the certificate must be "Microsoft RSA SChannel Cryptographic Provider".

Got all that? An ADCS template is the easiest way to set this up for consistent certificate creation. Those "should not contain" bits are important to helping lock down the certificate and preventing mis-use by a bad actor.

### Verify the Certificate
Once the certificate is installed on the node, you can run this - on the node - to verify the certificate and get its thumbprint:

```
function Get-LocalEncryptionCertificateThumbprint 
{ 
    (dir Cert:\LocalMachine\my) | %{
        # Verify the certificate is for Encryption and valid 
        if ($_.PrivateKey.KeyExchangeAlgorithm -and $_.Verify()) 
        { 
            return $_.Thumbprint 
        } 
    } 
}
```

### Defining a Configuration Data Block
Check this out:

```
$ConfigData= @{ 
    AllNodes = @(     
            @{  
                NodeName = "targetNode" 
                CertificateFile = "C:\publicKeys\targetNode.cer" 
                Thumbprint = "AC23EA3A9E291A75757A556D0B71CBBF8C4F6FD8" 
            }; 
        );    
    }
```

This Configuration Data block defines the node ("targetNode" is the computer name, here), the location of the certificate file (public key only) on the authoring computer, and the certificate thumbprint. Obviously, you can have a ton of these, one for each node. And yes, it really should be one for each node - sharing certificates is like sharing contraceptives. Ew.

### The Configuration Script
Whatever password you need to encrypt should be typed as a [PSCredential] data type - this triggers the encryption behavior during MOF creation. For example:

```
configuration CredentialEncryptionExample 
{ 
    param( 
        [Parameter(Mandatory=$true)] 
        [ValidateNotNullorEmpty()] 
        [PsCredential] $credential 
        ) 


    Node $AllNodes.NodeName 
    { 
        File exampleFile 
        { 
            SourcePath = "\\Server\share\path\file.ext" 
            DestinationPath = "C:\destinationPath" 
            Credential = $credential 
        } 
    } 
}
```

See where we created the credential? PowerShell will prompt for it when you run this, and you can see where it's used ($credential) in the configuration. You need to run this and pass in the Configuration Data block:

```
CredentialEncryptionExample -ConfigurationData $ConfigData -OutputPath .\CredentialEncryptionExample
```

So, we ran the configuration (I'd normally have this at the end of the script containing the configuration), and passed in the $ConfigData block I created earlier (which you might read from a separate file). This will generate a MOF for each node in $ConfigData. Because the $ConfigData has the local (on the authoring system) path for the certificate public key, and because it also has the thumbprint, it'll all just magically work.

### Sharing Certificates 
We made a fairly strong statement earlier about not sharing certificates, and we hold to this statement, but there are some good reasons for considering sharing certificates. Say you have a web server farm, and all of those servers are to be configured identically, and you're using Configuration Names for the configuration.  You have one MOF file to configure all the servers in the web farm. You can't deploy that configuration to multiple nodes that each have a different certificate. In that case, it may make sense to use one certificate for the web server configuration and deploy that certificate to all the web servers.

Keep in mind that you could choose to go all the way to the opposite extreme and just use a single certificate for all servers in your organization, and that's a design decision with some serious tradeoffs.  On one hand, it makes certificate management really easy, but on the other hand, if your private key gets compromised or the certificate expires without you knowing, you're probably going to have a really bad day.  

### Certificate Management
So... how do you manage all these deployed certificates? Well... that's still an evolving story. Honestly. A bunch of us at DevOps Camp (an event I run in August in Las Vegas for about 20 folks; keep an eye on DonJones.com for dates) have been fooling with ideas for automating installation, dealing with expirations and renewals, and so on, especially in non-domain environments. As practices emerge and are refined for this, I'll loop back and include that information in this book.

## PSDSCRunAsCredential
This is a universal property that, as of WMF 5.0, can be used with any resource. When a configuration is run, the LCM will launch the resource using the provided credential. You should follow the guidelines above for encrypting those credentials in the MOF!

Note that in WMF 5.0, you can only use PSRunAsCredential with "real" resources - that is, ones that are an actual compiled, class-based, or function-based resource module. You cannot use PSRunAsCredential with a _composite_ resource. However, that was changed in WMF 5.1, and you can pass a PSDSCRunAsCredential to a composite resource as well. The entire composite resource will run using the credential you provide. However, if the composite resource _itself_ specifies alternate credentials, then the LCM will explode and your configuration won't run.

## Digital Signing
This feature was added in WMF 5.1, and allows you to digitally sign configuration (MOF) files. If a node pulls a configuration from a pull server, and the signature isn't valid, the LCM will refuse to run the configuration. _This is a very good idea to implement_, because it helps protect against a compromised pull server. I know, you think, "we have firewalls, nobody can get to our pull server," to which I say, "defense in depth, man," and, "that's what the US Office of Personnel Management thought until June 2016." Sign MOFs using Set-AuthenticodeSignature _on a machine running WMF 5.1_ (the cmdlet in previous versions can't sign MOFs). 

Modules can also be signed. You have to start by creating a _catalog_ for the module, which is accomplished by the New-FileCatalog cmdlet. Sign the catalog file using Set-AuthenticodeSignature - again, this all works only in WMF 5.1 and later - and put the catalog file inside the module folder. If the module is named OurTools, then the catalog filename would be OurTools.cat, and it would live in the OurTools module folder.\

Then you have to tell the LCM to expect signatures - see the chapter on configuring the LCM for that. Signatures are validated on a per-pull-server basis, and only when the LCM downloads files; it does not re-validate signatures when performing a consistency check. You can also configure the LCM to expect signatures on pushed configurations.
