# DSCResources #
### Custom DSC resource module for Microsoft Monitoring Agent by [PowerShell Magazine](http://www.powershellmagazine.com "PowerShell Magazine"). ###

----------

Microsoft Monitoring Agent (MMA) is used to connect target systems to System Center Operations Manager or directly to Azure Operational Insights. This DSC resource module intends to build custom resources for installing and configuring MMA on target systems.

At present, there are four resources available within this module.

- [cMMAgentInstall](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentInstall) is used to install Microsoft Monitoring Agent.
- [cMMAgentProxyName](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyName) is used to add or remove the proxy URL for the Microsoft Monitoring Agent configuration.
- [cMMAgentProxyCredential](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyCredential) is used to add, modify, or remove the credentials that need to be used to authenticate to a proxy configured using cMMAgentProxyName resource.
- [cMMAgentOpInsights](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentOpInsights) is used to enable or disable Azure Operational Insights within the Microsoft Monitoring Agent. This can also be used to update the WorkspaceID and WorkspaceKey for connecting to Azure Operational Insights.

I could have combined the resources into just a couple of them but that increases the complexity of the resource module. Therefore, I decided to go much granular and divide these into multiple resources. For example, the [cMMAgentProxyCredential](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyCredential) resource lets you not just add or remove credentials but also update the credentials, if required.

####Using cMMAgentInstall resource####
The [cMMAgentInstall](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentInstall) is a composite DSC resource. This can be used to install MMAgent in an unattended manner. Behind the scenes, this resource uses the Package DSC resource.

![](http://i.imgur.com/tkrhB01.png)

The *Path* property is used to specify a local folder path where MMAgent installer is stored. This can also be the HTTP location for downloading an installer. For example, the following function can be used to get the redirected URL from fwlink on Microsoft site.

    Function Get-TrueURL {   
       Param (
          [parameter(Mandatory)]
          $Url
       )
       $req = [System.Net.WebRequest]::Create($url)
       $req.AllowAutoRedirect=$false
       $req.Method="GET"
    
       $resp=$req.GetResponse()
       if ($resp.StatusCode -eq "Found") {
          return $resp.GetResponseHeader("Location")
       }   
       else {
          return $resp.responseURI
       }
    }

The fwlink to download MMAgent installer is http://go.microsoft.com/fwlink/?LinkID=517476.

    $TrueUrl = Get-TrueUrl -Url 'http://go.microsoft.com/fwlink/?LinkID=517476'

Now, this redirected URL can be given as value can be given as the value of *Path* property. The Package resource downloads the installer before it attempts to install it on the target system.

The *WorkspaceID* and the *WorkspaceKey* can be retrieved from the [Azure Operational Insights preview portal](https://preview.opinsights.azure.com) by navigating to Servers & Usage -> Configure.

![](http://i.imgur.com/k2Q7q9j.png)

Here is a configuration script that uses the cMMAgentInstall composite resource.

    Configuration MMASetup {
       Import-DscResource -Module cMMAgent -Name cMMAgentInstall
       cMMAgentInstall MMASetup {
          Path = 'C:\OpInsights\MMASetup-AMD64.exe'
          Ensure = 'Present'
          WorkspaceID = 'your-Workspace-Id'
          WorkspaceKey = 'Your-Workspace-Key'
       }
    }
    
    MMASetup

####Using cMMAgentProxyName resource####
The [cMMAgentProxyName](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyName) resource can be used to add or remove the proxy URL to the Microsoft Monitoring Agent configuration.

![](http://i.imgur.com/TKThxSa.png)

The only property here is the *ProxyName* property. For this, you need to specify a proxy URL (either HTTP or HTTPS) with any port number as required. The following configuration script demonstrates this usage.

    cMMAgentProxyName MMAgentProxy {
        ProxyName = 'https://moxy.us.dell.com:3128'
        Ensure = 'Present'
    }

####Using cMMAgentProxyCredential resource####
The [cMMAgentProxyName](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyName) resource only lets you configure the proxy URL that needs to be used to connect to the Azure Operational Insights service. However, if the proxy requires authentication, you can use the [cMMAgentProxyCredential](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentProxyCredential) resource configure the same.

![](http://i.imgur.com/jRuGPa0.png)

When I was writing this resource module, there was a challenge in having only a *PSCredential* type property as the Key property. The schema MOF was failing validation and therefore forced me to separate it into *ProxyUserName* and *ProxyUserPassword* properties. The *ProxyUserPassword* is of type PSCredential and you can use the *Get-Credential* cmdlet to supply that. The user name you supply as a part of the PSCredential will not be used. Instead, the value supplied as an argument for the *ProxyUserName* property will be used.

Note that DSC best practices recommend that you encrypt the credentials used in a configuration script. So, ideally, you should use certificates to encrypt and decrypt the credentials. For test and development purposes, however, you can use the plain text passwords and this can be achieved by using DSC configuration data. The following example demonstrates this.

    $ProxyUserPassword = Get-Credential
    
    $ConfigData = @{
       AllNodes = @(
          @{ NodeName = '*'; PsDscAllowPlainTextPassword = $true },
          @{ NodeName = 'WMF5-1' }
       )
    }
    
    Configuration MMAgentConfiguration {
       Import-DscResource -ModuleName cMMAgent
       Node $AllNodes.NodeName {
          cMMAgentProxyCredential MMAgentProxyCred {
             ProxyUserName = 'ravikanth'
             ProxyUserPassword = $ProxyUserPassword
             Ensure = 'Present'
          }
       }
    }

Once you set the proxy credentials, if you need to change the password alone within the credentials, you can use the *UpdateCredential* property to force that change. When using *UpdateCredential* property, setting Ensure=Absent is not valid.

    $NewProxyUserPassword = Get-Credential
    
    $ConfigData = @{
       AllNodes = @(
          @{ NodeName = '*'; PsDscAllowPlainTextPassword = $true },
          @{ NodeName = 'WMF5-1' }
       )
    }
    
    Configuration MMAgentConfiguration {
       Import-DscResource -ModuleName cMMAgent
       Node $AllNodes.NodeName {
          cMMAgentProxyCredential MMAgentProxyCred {
             ProxyUserName = 'ravikanth'
             ProxyUserPassword = $NewProxyUserPassword
             UpdateCredential = $True
             Ensure = 'Present'
          }
       }
    }

####Using cMMAgentOpInsights resource####
The [cMMAgentInstall](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentInstall) resource, by default, enables the Microsoft Monitoring Agent for Azure Operational Insights. This resource also configures the WorkspaceID and the WorkspaceKey required to connect to the Operational Insights service. 

The [cMMAgentOpInsights](https://github.com/rchaganti/DSCResources/tree/master/cMMAgent/DSCResources/cMMAgentOpInsights) resource can be used to update the *WorkspaceID* and *WorkspaceKey*, if required and also to disable Azure Operational Insights within the Microsoft Monitoring Agent.

![](http://i.imgur.com/JptPoQB.png)

By setting both the *WorkspaceID* and the *WorkspaceKey* properties and *Ensure* to *Present*, you can enable Azure Operational Insights and update the WorkspaceID and WorkspaceKey.

        cMMAgentOpInsights MMAgentOpInsights {
            WorkspaceID = 'your-Workspace-ID'
            WorkspaceKey = 'your-Workspace-Key'
            Ensure = 'Absent'
        }

By setting *Ensure* to *Absent* along with required properties, Azure Operational Insights can be disable for the Microsoft Monitoring Agent.

        cMMAgentOpInsights MMAgentOpInsights {
            WorkspaceID = 'your-Workspace-ID'
            WorkspaceKey = 'your-Workspace-Key'
            Ensure = 'Absent'
        }

If you need to update only the WorkspaceKey, you can do that using the *UpdateWorkspace* property.

        cMMAgentOpInsights MMAgentOpInsights {
            WorkspaceID = 'your-Workspace-ID'
            WorkspaceKey = 'your-new-Workspace-Key'
            UpdateWorkspace = $true
        }