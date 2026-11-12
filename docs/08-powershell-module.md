# 08 – PowerShell module

`powershell\EndpointManagementApi` – one cmdlet per API route (list in [09](09-api-reference.md)). Runs in Windows PowerShell 5.1, PowerShell 7 and WinPE (task sequences). No dependencies.

The module is the convenient way to call the API from PowerShell. It is not required: any client that can get an Entra ID token and send HTTPS requests can call the API directly ([10](10-clients.md)).

## What the module does for you

| | |
|---|---|
| Sign-in | client credentials of the client's own app registration, with a certificate (recommended) or a client secret |
| Token | cached and renewed automatically 60 s before it expires |
| Answers | returns the response object: `.success`, `.data` (the answer: `$true`, an object, a list) and `.timestamp` |
| Errors | throws `EndpointManagement API POST /api/… failed (HTTP 403): Access denied. Required role(s): ConfigMgr.ReadWrite.` |
| Bodies | sent as JSON with the property names of the API (`UserName`, `TargetOU`, …) |
| Names in the address | every computer, user, group, collection or sequence name is URL-encoded as one path segment: a `/`, `?`, `#` or `%` stays part of the name; `.` and `..` are refused, also as a part between `/` or `\` (`a/../b`), since Windows PowerShell 5.1 can undo the encoding |
| Address | the token is only sent to an `https://` `BaseUrl`, and with `-AllowedHosts` only to those hosts |

## Install

Copy the folder `EndpointManagementApi` to a module path, e.g. `C:\Program Files\WindowsPowerShell\Modules\`, or import it from any folder:

```powershell
Import-Module .\EndpointManagementApi
```

The installer also puts it under `C:\Program Files\EndpointManagementAPI\powershell\` on the API server.

## Sign in

```powershell
# Certificate in CurrentUser\My or LocalMachine\My, uploaded to the client's app registration
Initialize-Api -BaseUrl 'https://endpoint-management.contoso.local' -TenantId '<tenant id>' `
               -ClientId '<client app id>' -ApiAppId '<EndpointManagement.Service app id>' `
               -CertificateThumbprint '<thumbprint>'

# or with a client secret, best as SecureString (a plain string works too)
$secret = Read-Host 'Client secret' -AsSecureString
Initialize-Api -BaseUrl 'https://sccm01.contoso.local/EndpointManagementAPI' -TenantId '<tenant id>' `
               -ClientId '<client app id>' -ApiAppId '<EndpointManagement.Service app id>' `
               -ClientSecret $secret
```

The module keeps the secret as SecureString for the session and turns it into text only for the token request.

| Parameter | Value |
|---|---|
| `BaseUrl` | address of the API without `/api`, e.g. `https://sccm01.contoso.local/EndpointManagementAPI`. Must be `https://`; the token is sent only there |
| `TenantId` | Entra tenant ID |
| `ClientId` | client ID of **this client's** app registration (e.g. `EndpointManagement.Client.TaskSequence`) |
| `ApiAppId` | client ID of `EndpointManagement.Service` – the module requests `api://<ApiAppId>/.default` |
| `CertificateThumbprint` / `ClientSecret` | credential of the client app registration |
| `AllowedHosts` | optional: host names `BaseUrl` may have, e.g. `'sccm01.contoso.local'`; any other host stops `Initialize-Api` |

## Examples

```powershell
(Test-ConfigMgrDevice -ComputerName 'CH-NB-04711').data                 # True / False
(Get-AdComputerInfo -ComputerName 'CH-NB-04711').data | Format-List
Add-ConfigMgrDeviceToCollection -ComputerName 'CH-NB-04711' -CollectionName 'Apps - Office'
Send-ConfigMgrStatusMessage -ComputerName 'CH-NB-04711' -Component 'OSD' -Description 'Apply OS done'   # not SMS_… or CCM…
Get-IntuneDeviceCompliance -ComputerName 'CH-NB-04711'
Get-ApiHealth
```

Autopilot, Intune actions and device attributes:

```powershell
Set-IntuneAutopilotGroupTag -ComputerName 'CH-NB-04711' -GroupTag 'Bern-Standard'   # also takes a serial number
Remove-IntuneAutopilotDevice -ComputerName 'PF4XK2J1'                              # serial number before a hardware swap
Invoke-IntuneRemediation -ComputerName 'CH-NB-04711' -RemediationName 'Clear Teams cache'
Invoke-IntuneDefenderScan -ComputerName 'CH-NB-04711' -FullScan
Rename-IntuneDevice -ComputerName 'CH-NB-04711' -NewName 'CH-NB-04712'
(Get-IntuneDetectedApps -ComputerName 'CH-NB-04711').data | Where-Object displayName -like '*Teams*'

Set-EntraDeviceExtensionAttribute -ComputerName 'CH-NB-04711' -Number 1 -Value 'Standard-Client'   # -Value '' clears it
Disable-EntraDevice -ComputerName 'CH-NB-04711'
Set-AdComputerAttribute -ComputerName 'CH-NB-04711' -AttributeName 'extensionAttribute3' -Value 'Bern'
New-AdComputerAccount -ComputerName 'CH-NB-05000' -TargetOU 'OU=Notebooks,OU=Clients,DC=contoso,DC=local'
```

Secrets need roles of their own (`LAPS.Read`, `AD.BitLocker.Read`, `Entra.BitLocker.Read`); each call is logged as a critical action:

```powershell
(Get-AdLapsPassword -ComputerName 'CH-NB-04711').data.password
(Get-EntraLapsPassword -ComputerName 'CH-NB-04711').data
$id = (Get-EntraBitLockerKeyIds -ComputerName 'CH-NB-04711').data | Select-Object -First 1
(Get-EntraBitLockerKey -ComputerName 'CH-NB-04711' -KeyId $id).data
Reset-IntuneLapsPassword -ComputerName 'CH-NB-04711'                                 # after the password was used
```

## Task sequences

`Initialize-ApiFromTsVariables` reads the sign-in values from task sequence variables:

| Variable | |
|---|---|
| `API_BASE_URL`, `API_TENANT_ID`, `API_CLIENT_ID`, `API_APP_ID` | required |
| `API_CLIENT_SECRET` | required unless `API_CERT_THUMBPRINT` is set |
| `API_CERT_THUMBPRINT` | optional, certificate instead of secret |

Set `API_CLIENT_SECRET` as a **hidden** task sequence variable (*Do not display this value*). In WinPE there is usually no certificate: use the task sequence app registration with only the roles the task sequence needs and a secret with a short lifetime ([10](10-clients.md)).

**Where to define the `API_*` variables:** in a *Set Task Sequence Variable* step of the task sequence (or in the task sequence's own variables), not as collection variables. A device variable overrides a collection variable of the same name, and a device variable is something a client of this API could set. The API refuses `API_*` device variables (`configMgr.reservedVariables`), and the module checks the address before it sends a token:

- `BaseUrl` must be an `https://` address without user name or password, otherwise `Initialize-Api` stops.
- `-AllowedHosts` limits it to known host names. Put it on the command line of the step, where no variable can change it.

```powershell
Import-Module "$PSScriptRoot\EndpointManagementApi"
Initialize-ApiFromTsVariables -AllowedHosts 'endpoint-management.contoso.local'
Register-ConfigMgrDevice -ComputerName $env:COMPUTERNAME -BiosGuid (Get-CimInstance Win32_ComputerSystemProduct).UUID
```
