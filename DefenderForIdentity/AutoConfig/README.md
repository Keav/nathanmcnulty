# Automated Configuration

This is a collection of commands that will help automate the configuration of the Defender for Identity settings. To use this, you must obtain the sccauth value and xsrf-token value from the browser and use it to create cookies and headers for our API calls. This is because we are using an internal API to configure settings, and there isn't a public way to get the right tokens.

## Table of Contents

[Setting up our session and cookies](README.md#setting-up-our-session-and-cookies)

[Creating the workspace](README.md#creating-the-workspace)

[General - Sensors](README.md#sensors)

[General - Directory services accounts](README.md#directory-services-accounts)

[General - Manage action accounts](README.md#roles)

[General - VPN](README.md#vpn)

[General - Adjust alert threshholds](README.md#adjust-alert-threshholds)

[General - About](README.md#about)

[Entity tags - Sensitive](README.md#sensitive)

[Entity tags - Honeytoken](README.md#honeytoken)

[Entity tags - Exchange server](README.md#exchange-server)

[Actions and exclusions - Global excluded entities](README.md#global-excluded-entities)

[Actions and exclusions - Exclusions by detection rule](README.md#exclusions-by-detection-rule)

[Notifications - Health issues notifications](README.md#health-issues-notifications)

[Notifications - Alert notifications](README.md#alert-notifications)

[Notifications - Syslog notifications](README.md#syslog-notifications)

## Setting up our session and cookies

First, we need to create a WebRequestSession object contaning the sccauth and xsrf cookies copied from the browser and headers with the xsrf token. To get this, open Developer Tools in your browser and make sure the Network tab is set to preserve logs, then log into security.microsoft.com. Search for **apiproxy** and select a request.

![img](./img/sccauth-1.png)

Under headers, scroll down under the cookies section, copy the value after sccauth (it is very long) all the way to the next semicolon and save it into the $sccauth variable. Now do the same for xsrf-token and save it into the $xsrf variable.

![img](./img/sccauth-2.png)

Now we can create a session with those cookies:

```powershell
# Copy sccauth from the browser
$sccauth = Read-Host -Prompt "Enter sccauth cookie value" -AsSecureString
if ($sccauth.Length -ne 2368) { Write-Warning "sccauth was $(sccauth.Length) characters and may be incorrect" }

# Copy xsrf token from the browser
$xsrf = Read-Host -Prompt "Enter xsrf cookie value" -AsSecureString
if ($xsrf.Length -ne 347) { Write-Warning "xsrf was $($xsrf.Length) characters and may be incorrect" }

# Create session and cookies
$session = New-Object Microsoft.PowerShell.Commands.WebRequestSession
$session.Cookies.Add((New-Object System.Net.Cookie("sccauth", "$($sccauth | ConvertFrom-SecureString -AsPlainText)", "/", "security.microsoft.com")))
$session.Cookies.Add((New-Object System.Net.Cookie("XSRF-TOKEN", "$($xsrf | ConvertFrom-SecureString -AsPlainText)", "/", "security.microsoft.com")))

# Set the headers to include the xsrf token
[Hashtable]$Headers=@{}
$headers["X-XSRF-TOKEN"] = [System.Net.WebUtility]::UrlDecode($session.cookies.GetCookies("https://security.microsoft.com")['xsrf-token'].Value)

```

With this complete, we can now make requests to the internal API :)

## Creating the workspace

We can check and see if the Defender for Identity workspace has been created yet, and if not, create it. For new deployments, this will be important to kick off provisioining and check before we attempt to configure the service ;)

```powershell
# Check if workspace exists
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspaces/isWorkspaceExists/" -ContentType "application/json" -WebSession $session -Headers $headers

# Next time I spin up a new tenant, I'll document creation, lol

```
## General

### Sensors

This is where we can check health of existing sensors and download new sensors for installation

```powershell
# Check how many DCs exist and are being covered
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/sensors/domainControllerCoverage" -ContentType "application/json" -WebSession $session -Headers $headers

# Get list of sensors
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/sensors" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get access key
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/sensorDeploymentAccessKey" -ContentType "application/json" -WebSession $session -Headers $headers

# Get sensor download
$url = Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/sensors/deploymentPackageUri" -ContentType "application/json" -WebSession $session -Headers $headers

Invoke-WebRequest -Uri $url -OutFile "$env:USERPROFILE\Downloads\Azure ATP Sensor Setup.zip"

```

To edit sensor settings, we can do the following:

```powershell
# Get sensor
$name = "sml-dc01"
$sensor = (Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/sensors" -ContentType "application/json" -WebSession $session -Headers $headers).value | Where-Object { $_.Name -eq $name }

# Here you can modify Description and whether the network adapter is enabled or not
$body = @{
  Description = "Description"
  NetworkAdapters = @(@{
    Id = $sensor.Settings.NetworkAdapters.Id
    IsEnabled = $true
    Name = $sensor.Settings.NetworkAdapters.Name
  })
  DomainControllerDnsNames = @($sensor.Settings.DomainControllerDnsNames)
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "PUT" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/sensors/$($sensor.Id)/settings" -ContentType "application/json" -Body $body -WebSession $session -Headers $headers

```

To enable/disable delayed updates, we can do the following:

```powershell
# Get sensor
$name = "sml-dc01"
$sensor = (Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/sensors" -ContentType "application/json" -WebSession $session -Headers $headers).value | Where-Object { $_.Name -eq $name }

# $true enables delayed deployment, $false disables delayed deployment
$body = @{ IsDelayedDeploymentEnabled = $false } | ConvertTo-Json

Invoke-RestMethod -Method "PUT" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/sensors/$($sensor.Id)/settings" -ContentType "application/json" -Body $body -WebSession $session -Headers $headers

```

I did not document deleting a sensor here for a few reasons, but it is possible to mass delete sensors... :)

### Directory services accounts

Directory services accounts

```powershell
# Get a list of directory services accounts
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/directoryServices" -ContentType "application/json" -WebSession $session -Headers $headers).value

```

To add a directory services account:

```powershell
# Add a directory services accounts
$body = @{
  Id = ""
  AccountName = "gmsa-mdi-ds"
  DomainDnsName = "sharemylabs.com"
  AccountPassword = $null
  IsGroupManagedServiceAccount = $true
  IsSingleLabelAccountDomainName = $false
} | ConvertTo-Json

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/directoryServices" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers -AllowInsecureRedirect

```

To delete a directory services account:

```powershell
# Delete a directory services accounts
$body = @{ id = "gmsa-mdi-ds@sharemylabs.com" } | ConvertTo-Json

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/directoryServices/delete" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### Manage action accounts

Manage action accounts


```powershell
# Get configuration for action accounts
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/remediationActions/configuration" -ContentType "application/json" -WebSession $session -Headers $headers

```

Enable using local SYSTEM account for remediation action:

```powershell
# Turn off using local system for action account
$body = @{ IsRemediationWithLocalSystemEnabled = $true }

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/remediationActions/configuration" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers
```

Turn off system account for remediation and manually configure a gMSA (not recommended):

```powershell
# Add action accout
$body = @{ 
  Id = ""
  AccountName = "gmsa-mdi-action"
  DomainDnsName = "sharemylabs.com"
  AccountPassword = $null
  IsGroupManagedServiceAccount = $true
  IsSingleLabelAccountDomainName = $false 
}

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/EntityRemediatorCredentials" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Turn off using local system for action account
$body = @{ IsRemediationWithLocalSystemEnabled = $false }

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/remediationActions/configuration" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### VPN

VPN

```powershell
# Get current configuration
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/mtp/vpnConfiguration/" -ContentType "application/json" -WebSession $session -Headers $headers

```

Enable RADIUS accounting and save shared secret:

```powershell
# Enable and configure RADIUS accounting shared secret
$body = @{
  IsRadiusEventListenerEnabled = $true
  RadiusEventListenerSharedSecret = "secretValue"
} | ConvertTo-Json
Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/mtp/vpnConfiguration/" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

Turn off RADIUS accounting:

```powershell
# Disable RADIUS accounting
$body = @{
  IsRadiusEventListenerEnabled = $false
  RadiusEventListenerSharedSecret = ""
} | ConvertTo-Json
Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/mtp/vpnConfiguration/" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### Adjust alert threshholds

Adjust alert threshholds

```powershell
$response = Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/alertthresholds" -ContentType "application/json" -WebSession $session -Headers $headers

# Check if Recommended test mode is enabled
$response.IsRecommendedTestModeEnabled

# Check alert thresholds
$response.AlertThresholds

```

To change a threshold:

```powershell
# Get current configuration
$response = Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/alertthresholds" -ContentType "application/json" -WebSession $session -Headers $headers

# Review thresholds
$response.AlertThresholds

# Change thresholds
$response.AlertThresholds[1].Threshold = "Low"
$response.AlertThresholds[6].Threshold = "Medium"

# Save body
$body = $response | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/alertthresholds" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### About

This is some basic data about the workspace and licenses

```powershell
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/mtp/applicationData" -ContentType "application/json" -WebSession $session -Headers $headers

```

## Entity tags

### Sensitive

Sensitive accounts are used to identify high-value assets which are used by some detections. The lateral movement path also relies on an entity's sensitivity status. There are three types of sensitive entity tags - users, devices, and groups. [Learn more](https://aka.ms/MDI/EntityTags)

For users, remember that these identities could be from non-Entra sources, so we have to use the object IDs from Defender for Identity.

```powershell
# Get list of current sensitive users (first 100, adjust filter if you want more)
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27User%27%20and%20TagTypes%20has%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of users (first 100, adjust filter if you want more)
$body = @{ SearchType = "User" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search users by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "User"
  Filter = "Nathan"
} | ConvertTo-Json

$users = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $users.Add($_.Id) | Out-Null }

# Add sensitive tag for users
$body = @{
    EntitiesType = "User"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove sensitive tag for users
$body = @{
    EntitiesType = "User"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For devices:

```powershell
# Get list of current sensitive devices (first 100, adjust filter if you want more)
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27Computer%27%20and%20TagTypes%20has%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of devices (first 100, adjust filter if you want more)
$body = @{ SearchType = "Computer" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search devices by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Computer"
  Filter = "sml"
} | ConvertTo-Json

$computers = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $computers.Add($_.Id) | Out-Null }

# Add sensitive tag for computers
$body = @{
    EntitiesType = "Computer"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove sensitive tag for computers
$body = @{
    EntitiesType = "Computer"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For groups:

```powershell
# Get list of current sensitive groups (first 100, adjust filter if you want more)
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27Group%27%20and%20TagTypes%20has%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of groups (first 100, adjust filter if you want more)
$body = @{ SearchType = "Group" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search group by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Group"
  Filter = "sml"
} | ConvertTo-Json

$groups = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Sensitive%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $groups.Add($_.Id) | Out-Null }

# Add sensitive tag for groups
$body = @{
    EntitiesType = "Group"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $groups
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove sensitive tag for groups
$body = @{
    EntitiesType = "Group"
    TagType = @("Sensitive")
    SecurityPrincipalIds = $groups
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### Honeytoken

Honeytoken accounts are used as traps for malicious actors. Any authentication associated with these honeytoken accounts triggers an alert. There are two types of honeytoken accounts we can create - users and devices. [Learn more](https://aka.ms/MDI/EntityTags)

For users:

```powershell
# Get list of current users tagged as honeytokens
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27User%27%20and%20TagTypes%20has%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of users (first 100, adjust filter if you want more)
$body = @{ SearchType = "User" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search users by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "User"
  Filter = "Albert"
} | ConvertTo-Json

$users = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $users.Add($_.Id) | Out-Null }

# Tag users as honeytokens
$body = @{
    EntitiesType = "User"
    TagType = @("Honeytoken")
    SecurityPrincipalIds = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Untag users as honeytokens
$body = @{
    EntitiesType = "User"
    TagType = @("Honeytoken")
    SecurityPrincipalIds = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For devices:

```powershell
# Get list of devices tagged as honeytokens
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27Computer%27%20and%20TagTypes%20has%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of devices (first 100, adjust filter if you want more)
$body = @{ SearchType = "Computer" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search devices by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Computer"
  Filter = "sml"
} | ConvertTo-Json

$computers = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $computers.Add($_.Id) | Out-Null }

# Tag devices as honeytokens
$body = @{
    EntitiesType = "Computer"
    TagType = @("Honeytoken")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Untag devices as honeytokens
$body = @{
    EntitiesType = "Computer"
    TagType = @("Honeytoken")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### Exchange servers

Tag devices as Exchange servers. Microsoft Defender for Identity considers Exchange servers as high-value assets and automatically tags them as Sensitive. [Learn more](https://aka.ms/MDI/EntityTags)

```powershell
# Get list of current exchange servers (first 100, adjust filter if you want more)
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/TaggedSecurityPrincipals?`$filter=Type%20eq%20%27Computer%27%20and%20TagTypes%20has%20%27ExchangeServer%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of devices (first 100, adjust filter if you want more)
$body = @{ SearchType = "Computer" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27ExchangeServer%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search devices by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Computer"
  Filter = "sml"
} | ConvertTo-Json

$computers = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27ExchangeServer%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $computers.Add($_.Id) | Out-Null }

# Tag devices as an Exchange server
$body = @{
    EntitiesType = "Computer"
    TagType = @("ExchangeServer")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Untag users as an Exchange server
$body = @{
    EntitiesType = "Computer"
    TagType = @("ExchangeServer")
    SecurityPrincipalIds = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/tagging" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

## Actions and exclusions

### Global excluded entities

This section is to exclude users, domains, devices, and IP addresses from *all* detection rules. It is a best practice to use per-rule exclusions and/or Alert tuning instead of globally excluding entities.

For users:

```powershell
# Get list of current excluded users
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/ExclusionEntityDatas/Global?`$filter=ExclusionType%20eq%20%27User%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search users by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "User"
  Filter = "Albert"
} | ConvertTo-Json

$users = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$filter=TagTypes%20ne%20%27Honeytoken%27&`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $users.Add($_.Id) | Out-Null }

# Add users
$body = @{
    ExclusionType = @("User")
    ExcludedEntityIdentifiers = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove users
$body = @{
    ExclusionType = @("User")
    ExcludedEntityIdentifiers = $users
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For domains:

```powershell
# Get list of current excluded domains
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/ExclusionEntityDatas/Global?`$filter=ExclusionType%20eq%20%27DomainName%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Add a domain
$body = @{
    ExclusionType = @("DomainName")
    ExcludedEntityIdentifiers = @("infection.monkey.sharemylabs.com","myc2.sharemylabs.com")
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove a domain
$body = @{
    ExclusionType = @("DomainName")
    ExcludedEntityIdentifiers = @("infection.monkey.sharemylabs.com","myc2.sharemylabs.com")
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For devices:

```powershell
# Get list of excluded devices
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/ExclusionEntityDatas/Global?`$filter=ExclusionType%20eq%20%27Computer%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get list of devices (first 100, adjust filter if you want more)
$body = @{ SearchType = "Computer" } | ConvertTo-Json

(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search devices by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Computer"
  Filter = "sml"
} | ConvertTo-Json

$computers = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $computers.Add($_.Id) | Out-Null }

# Add devices to global exclusion
$body = @{
    ExclusionType = @("Computer")
    ExcludedEntityIdentifiers = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove devices from global exclusion
$body = @{
    ExclusionType = @("Computer")
    ExcludedEntityIdentifiers = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

For IP Addresses:

```powershell
# Get list of current excluded IP addresses
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/ExclusionEntityDatas/Global?`$filter=ExclusionType%20eq%20%27Subnet%27&`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Add IP addresses
$body = @{
    ExclusionType = @("Subnet")
    ExcludedEntityIdentifiers = @("10.10.10.10","1.1.1.1")
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove IP addresses
$body = @{
    ExclusionType = @("Subnet")
    ExcludedEntityIdentifiers = @("10.10.10.10","1.1.1.1")
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/Global" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

### Exclusions by detection rule

Occasionally we need to exclude users, devices, or IPs from creating alerts for specific detection rules. The most common example will be Suspected DCSync attack (replication of directory services) which is triggered by the Entra Connect Sync server, so I will walk through discovering the detection rules metadata and create an exclusion for the Entra Connect Sync server for this rule.

This is how we can enumarate details about the detection rules, such as their internal names (SecurityAlertTypeName is like the Id which is used as an endpoint in the API), what types of exclusions they support (user, computer, or subnet), count of exclusions for each type, and localized names (the pretty display name we see in the GUI).

```powershell
# List all detection rules
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SecurityAlertExclusionDatas/?`$count=true&`$top=100&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search for detection rules by name (can see how we do OData filters!)
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SecurityAlertExclusionDatas/?`$filter=contains(tolower(TranslationData/Fallback),%27dcsync%27)&`$count=true&`$top=20&`$skip=0" -ContentType "application/json" -WebSession $session -Headers $headers).value

# Get details about the DirectoryServicesReplicationSecurityAlert detection rule
$SecurityAlertTypeName = "DirectoryServicesReplicationSecurityAlert"
(Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/odata/ExclusionEntityDatas/$SecurityAlertTypeName`?`$count=true&`$top=3&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value

# Search devices by name (first 100, adjust filter if you want more)
$body = @{
  SearchType = "Computer"
  Filter = "AADCONNECT"
} | ConvertTo-Json

$computers = New-Object System.Collections.ArrayList
(Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/odata/SearchSecurityPrincipals?`$count=true&`$top=100&`$skip=0" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers).value | Out-GridView -PassThru | ForEach-Object { $computers.Add($_.Id) | Out-Null }

# Add Entra Connect Servers as an exclusion
$body = @{
    ExclusionType = @("Computer")
    ExcludedEntityIdentifiers = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/$SecurityAlertTypeName" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

# Remove Entra Connect Servers as an exclusion
$body = @{
    ExclusionType = @("Computer")
    ExcludedEntityIdentifiers = $computers
} | ConvertTo-Json -Depth 4

Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/exclusion/$SecurityAlertTypeName" -Body $body -ContentType "application/json" -WebSession $session -Headers $headers

```

## Notifications

### Health issue notifications

I generally recommend adding a distribution list / mail enabled security group to receive helath issue notifications unless you plan to obtain alerts in a different way. Graph API now has these alerts which enables automation to send these via Teams chat / channel messages: https://learn.microsoft.com/en-us/graph/api/resources/healthmonitoring-overview?view=graph-rest-beta

```powershell
# Get current email addresses
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/healthNotifications/" -ContentType "application/json" -WebSession $session -Headers $headers

# Add an email address
Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/healthNotifications/" -Body '{"Email":"test@sharemylabs.com"}' -ContentType "application/json" -WebSession $session -Headers $headers -AllowInsecureRedirect

# Remove an email address
Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/healthNotifications/" -Body '{"Email":"test@sharemylabs.com"}' -ContentType "application/json" -WebSession $session -Headers $headers

```

### Alert notifications

It is recommended to use Defender XDR Alert notifications instead of alert notifications from Defender for Identity as it is more flexible and is the long term solution for incident/alert notifications. We can still configure these for now, so I'll document them here.

```powershell
# Get current email addresses
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/alertNotifications/" -ContentType "application/json" -WebSession $session -Headers $headers

# Add an email address
Invoke-RestMethod -Method "POST" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/alertNotifications/" -Body '{"Email":"test@sharemylabs.com"}' -ContentType "application/json" -WebSession $session -Headers $headers -AllowInsecureRedirect

# Remove an email address
Invoke-RestMethod -Method "DELETE" -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/alertNotifications/" -Body '{"Email":"test@sharemylabs.com"}' -ContentType "application/json" -WebSession $session -Headers $headers

```

### Syslog notifications

Only documenting how to check if syslog is still enabled and configured. All health and alert notifications should be retrived via API at this point rather than syslog.

```powershell
# Get current syslog config
Invoke-RestMethod -Uri "https://security.microsoft.com/apiproxy/aatp/api/workspace/configuration/syslog" -ContentType "application/json" -WebSession $session -Headers $headers

```