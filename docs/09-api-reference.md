# 09 – API reference

All 91 routes with the roles they need and the PowerShell cmdlet that calls them. The live description with request details is at `/api/docs` (OpenAPI); the admin UI lists the same under *Routes*.

Every answer uses the envelope `{ success, data?, error?, timestamp }` (`timestamp` = `yyyy-MM-ddTHH:mm:ssZ`, UTC). Exceptions: `/api/monitoring/health` (health summary) and `/api/monitoring/log` (XML).

| Status | Meaning |
|---|---|
| 200 | ok – `data` holds the answer, or is missing for actions |
| 400 | invalid route parameter or body (message names the field) |
| 401 | token missing, expired or not issued for this API |
| 403 | valid token, role missing – `Access denied. Required role(s): …`; or the role is there but the object is out of scope: protected group, group or OU outside the managed lists, domain controller, reserved task sequence variable ([06](06-configuration.md#scope-what-a-readwrite-role-may-reach)) |
| 404 | device, user, group, collection … not found |
| 409 | conflict: device already registered, collection busy, full policy reset on a device without an active ConfigMgr client, several objects with that name (Entra ID and Intune devices, groups, AD computers): the API does not pick one, delete the stale objects |
| 422 | the request would need too much work in the target system, e.g. more than 500 user collections to read for `…/applications/users/{userName}` |
| 429 | rate limit reached, or too many ConfigMgr requests of one caller at the same time – `Retry-After` header says how many seconds to wait ([07](07-operations.md#rate-limits)) |
| 500 | error in the target system (message in `error`, e.g. `Active Directory: insufficient access rights of the LDAP account (LDAP result 50).`); any other error only says `Internal error. Request ID …`, the details are in `app.log` |
| 503 | area turned off in the admin UI, or connector not available |

Every answer under `/api/` carries `Cache-Control: no-store`, so LAPS passwords, BitLocker keys and other answers are not kept by a browser or proxy.

Request bodies are JSON; property names are matched ignoring case (`UserName`, `userName`). Route parameters: computer names up to 64 characters (letters, digits, hyphens), user names up to 104 characters, group and collection names up to 256 characters.

## ConfigMgr – /api/configmgr

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| POST | `/api/configmgr/devices` | ConfigMgr.ReadWrite | `Register-ConfigMgrDevice` |
| GET | `/api/configmgr/devices/{computerName}/exists` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Test-ConfigMgrDevice` |
| DELETE | `/api/configmgr/devices/{computerName}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrDevice` |
| DELETE | `/api/configmgr/devices/bios-guid/{guid}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrDeviceByBiosGuid` |
| GET | `/api/configmgr/devices/{computerName}/primary-users` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Get-ConfigMgrPrimaryUsers` |
| POST | `/api/configmgr/devices/{computerName}/primary-users` | ConfigMgr.ReadWrite | `Add-ConfigMgrPrimaryUser` |
| DELETE | `/api/configmgr/devices/{computerName}/primary-users/{userName}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrPrimaryUser` |
| PUT | `/api/configmgr/devices/{computerName}/primary-users` | ConfigMgr.ReadWrite | `Sync-ConfigMgrPrimaryUsers` |
| GET | `/api/configmgr/devices/{computerName}/collections/{collectionName}/membership` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Get-ConfigMgrCollectionMembership` |
| POST | `/api/configmgr/devices/{computerName}/collections/{collectionName}` | ConfigMgr.ReadWrite | `Add-ConfigMgrDeviceToCollection` |
| DELETE | `/api/configmgr/devices/{computerName}/collections/{collectionName}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrDeviceFromCollection` |
| POST | `/api/configmgr/devices/{computerName}/osd-collections` | ConfigMgr.ReadWrite | `Add-ConfigMgrDeviceToOsdCollection` |
| DELETE | `/api/configmgr/devices/{computerName}/osd-collections/{advertisementId}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrDeviceFromOsdCollection` |
| DELETE | `/api/configmgr/devices/{computerName}/pxe` | ConfigMgr.ReadWrite | `Clear-ConfigMgrPxeFlag` |
| POST | `/api/configmgr/devices/{computerName}/restaging` | ConfigMgr.ReadWrite | `Invoke-ConfigMgrDeviceRestaging` |
| PUT | `/api/configmgr/devices/{computerName}/variables/{variableName}` | ConfigMgr.ReadWrite | `Set-ConfigMgrDeviceVariable` |
| DELETE | `/api/configmgr/devices/{computerName}/variables/{variableName}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrDeviceVariable` |
| PUT | `/api/configmgr/devices/{computerName}/variables` | ConfigMgr.ReadWrite | `Set-ConfigMgrDeviceVariables` |
| POST | `/api/configmgr/devices/{computerName}/status-messages` | ConfigMgr.ReadWrite | `Send-ConfigMgrStatusMessage` |
| GET | `/api/configmgr/usmt/{sourceComputer}/{destComputer}` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Get-ConfigMgrUsmtAssociation` |
| POST | `/api/configmgr/usmt` | ConfigMgr.ReadWrite | `Add-ConfigMgrUsmtAssociation` |
| DELETE | `/api/configmgr/usmt/{sourceComputer}/{destComputer}` | ConfigMgr.ReadWrite | `Remove-ConfigMgrUsmtAssociation` |
| POST | `/api/configmgr/devices/{computerName}/policy-refresh` | ConfigMgr.ReadWrite | `Invoke-ConfigMgrPolicyRefresh` |
| POST | `/api/configmgr/devices/{computerName}/policy-reset` | ConfigMgr.ReadWrite | `Invoke-ConfigMgrPolicyReset` |
| POST | `/api/configmgr/devices/{computerName}/hardware-inventory` | ConfigMgr.ReadWrite | `Invoke-ConfigMgrHardwareInventory` |
| POST | `/api/configmgr/collections/{collectionName}/refresh` | ConfigMgr.ReadWrite | `Update-ConfigMgrCollection` |
| GET | `/api/configmgr/devices/{computerName}/applications` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Get-ConfigMgrDeviceApplications` |
| GET | `/api/configmgr/devices/{computerName}/applications/users/{userName}` | ConfigMgr.Read or ConfigMgr.ReadWrite | `Get-ConfigMgrUserApplications` |

## Active Directory – /api/ad

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| GET | `/api/ad/computers/{computerName}/exists` | AD.Read or AD.ReadWrite | `Test-AdComputer` |
| DELETE | `/api/ad/computers/{computerName}` | AD.ReadWrite | `Remove-AdComputer` |
| PUT | `/api/ad/computers/{computerName}/ou` | AD.ReadWrite | `Move-AdComputer` |
| GET | `/api/ad/users/{userName}/groups` | AD.Read or AD.ReadWrite | `Get-AdUserGroups` |
| GET | `/api/ad/computers/{computerName}/info` | AD.Read or AD.ReadWrite | `Get-AdComputerInfo` |
| PUT | `/api/ad/computers/{computerName}/description` | AD.ReadWrite | `Set-AdComputerDescription` |
| GET | `/api/ad/computers/{computerName}/groups` | AD.Read or AD.ReadWrite | `Get-AdComputerGroups` |
| POST | `/api/ad/computers/{computerName}/groups/{groupName}` | AD.ReadWrite | `Add-AdComputerToGroup` |
| DELETE | `/api/ad/computers/{computerName}/groups/{groupName}` | AD.ReadWrite | `Remove-AdComputerFromGroup` |
| GET | `/api/ad/users/{userName}/exists` | AD.Read or AD.ReadWrite | `Test-AdUser` |
| GET | `/api/ad/users/{userName}/info` | AD.Read or AD.ReadWrite | `Get-AdUserInfo` |
| POST | `/api/ad/users/{userName}/groups/{groupName}` | AD.ReadWrite | `Add-AdUserToGroup` |
| DELETE | `/api/ad/users/{userName}/groups/{groupName}` | AD.ReadWrite | `Remove-AdUserFromGroup` |
| GET | `/api/ad/devices/{computerName}/bitlocker/key-ids` | AD.Read or AD.ReadWrite | `Get-AdBitLockerKeyIds` |
| GET | `/api/ad/devices/{computerName}/bitlocker/keys/{keyId}` | **AD.BitLocker.Read** | `Get-AdBitLockerKey` |
| GET | `/api/ad/computers/{computerName}/laps` | **LAPS.Read** | `Get-AdLapsPassword` |
| GET | `/api/ad/computers/{computerName}/laps/status` | AD.Read or AD.ReadWrite | `Get-AdLapsStatus` |
| PUT | `/api/ad/computers/{computerName}/enabled` | AD.ReadWrite | `Enable-AdComputerAccount`, `Disable-AdComputerAccount` |
| PUT | `/api/ad/computers/{computerName}/attributes/{attributeName}` | AD.ReadWrite | `Set-AdComputerAttribute` |
| POST | `/api/ad/computers` | AD.ReadWrite | `New-AdComputerAccount` |

## Intune – /api/intune

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| GET | `/api/intune/devices/{computerName}/exists` | Intune.Read or Intune.ReadWrite | `Test-IntuneDevice` |
| DELETE | `/api/intune/devices/{computerName}` | Intune.ReadWrite | `Remove-IntuneDevice` |
| GET | `/api/intune/devices/{computerName}/co-management` | Intune.Read or Intune.ReadWrite | `Get-IntuneCoManagementStatus` |
| PUT | `/api/intune/devices/{computerName}/primary-user` | Intune.ReadWrite | `Set-IntunePrimaryUser` |
| DELETE | `/api/intune/devices/{computerName}/primary-user` | Intune.ReadWrite | `Remove-IntunePrimaryUser` |
| PUT | `/api/intune/devices/{computerName}/category` | Intune.ReadWrite | `Set-IntuneDeviceCategory` |
| GET | `/api/intune/devices/{computerName}/info` | Intune.Read or Intune.ReadWrite | `Get-IntuneDeviceInfo` |
| GET | `/api/intune/devices/{computerName}/compliance` | Intune.Read or Intune.ReadWrite | `Get-IntuneDeviceCompliance` |
| POST | `/api/intune/devices/{computerName}/sync` | Intune.ReadWrite | `Sync-IntuneDevice` |
| POST | `/api/intune/devices/{computerName}/restart` | Intune.ReadWrite | `Restart-IntuneDevice` |
| POST | `/api/intune/devices/{computerName}/lock` | Intune.ReadWrite | `Lock-IntuneDevice` |
| POST | `/api/intune/devices/{computerName}/wipe` | **Intune.Wipe** | `Invoke-IntuneDeviceWipe` |
| POST | `/api/intune/devices/{computerName}/retire` | Intune.ReadWrite | `Invoke-IntuneDeviceRetire` |
| GET | `/api/intune/users/{userName}/devices` | Intune.Read or Intune.ReadWrite | `Get-IntuneUserDevices` |
| GET | `/api/intune/devices/{computerName}/autopilot` | Intune.Read or Intune.ReadWrite | `Get-IntuneAutopilotDevice` |
| PUT | `/api/intune/devices/{computerName}/autopilot/group-tag` | Intune.ReadWrite | `Set-IntuneAutopilotGroupTag` |
| DELETE | `/api/intune/devices/{computerName}/autopilot` | Intune.ReadWrite | `Remove-IntuneAutopilotDevice` |
| POST | `/api/intune/devices/{computerName}/laps/rotate` | Intune.ReadWrite | `Reset-IntuneLapsPassword` |
| POST | `/api/intune/devices/{computerName}/remediations` | Intune.ReadWrite | `Invoke-IntuneRemediation` |
| PUT | `/api/intune/devices/{computerName}/name` | Intune.ReadWrite | `Rename-IntuneDevice` |
| POST | `/api/intune/devices/{computerName}/defender-scan` | Intune.ReadWrite | `Invoke-IntuneDefenderScan` |
| GET | `/api/intune/devices/{computerName}/apps` | Intune.Read or Intune.ReadWrite | `Get-IntuneDetectedApps` |
| GET | `/api/intune/remediations` | Intune.Read or Intune.ReadWrite | `Get-IntuneRemediations` |

## Entra ID – /api/entra

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| POST | `/api/entra/groups/{groupName}/devices/{computerName}` | Entra.ReadWrite | `Add-EntraDeviceToGroup` |
| DELETE | `/api/entra/groups/{groupName}/devices/{computerName}` | Entra.ReadWrite | `Remove-EntraDeviceFromGroup` |
| POST | `/api/entra/groups/{groupName}/users/{samAccountName}` | Entra.ReadWrite | `Add-EntraUserToGroup` |
| DELETE | `/api/entra/groups/{groupName}/users/{samAccountName}` | Entra.ReadWrite | `Remove-EntraUserFromGroup` |
| GET | `/api/entra/devices/{computerName}/exists` | Entra.Read or Entra.ReadWrite | `Test-EntraDevice` |
| GET | `/api/entra/devices/{computerName}/info` | Entra.Read or Entra.ReadWrite | `Get-EntraDeviceInfo` |
| GET | `/api/entra/groups/{groupName}/exists` | Entra.Read or Entra.ReadWrite | `Test-EntraGroup` |
| GET | `/api/entra/groups/{groupName}/members` | Entra.Read or Entra.ReadWrite | `Get-EntraGroupMembers` |
| GET | `/api/entra/users/{samAccountName}/exists` | Entra.Read or Entra.ReadWrite | `Test-EntraUser` |
| GET | `/api/entra/users/{samAccountName}/info` | Entra.Read or Entra.ReadWrite | `Get-EntraUserInfo` |
| GET | `/api/entra/users/{samAccountName}/devices` | Entra.Read or Entra.ReadWrite | `Get-EntraUserDevices` |
| GET | `/api/entra/devices/{computerName}/extension-attributes` | Entra.Read or Entra.ReadWrite | `Get-EntraDeviceExtensionAttributes` |
| PUT | `/api/entra/devices/{computerName}/extension-attributes/{number}` | Entra.ReadWrite | `Set-EntraDeviceExtensionAttribute` |
| PUT | `/api/entra/devices/{computerName}/enabled` | Entra.ReadWrite | `Enable-EntraDevice`, `Disable-EntraDevice` |
| GET | `/api/entra/devices/{computerName}/laps` | **LAPS.Read** | `Get-EntraLapsPassword` |
| GET | `/api/entra/devices/{computerName}/laps/status` | Entra.Read or Entra.ReadWrite | `Get-EntraLapsStatus` |
| GET | `/api/entra/devices/{computerName}/bitlocker/key-ids` | Entra.Read or Entra.ReadWrite | `Get-EntraBitLockerKeyIds` |
| GET | `/api/entra/devices/{computerName}/bitlocker/keys/{keyId}` | **Entra.BitLocker.Read** | `Get-EntraBitLockerKey` |

## Monitoring – /api/monitoring

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| GET | `/api/monitoring/log` | Monitoring.Read | `Get-ApiLog` – XML built from `app.log` |
| GET | `/api/monitoring/health` | Monitoring.Read | `Get-ApiHealth` |

## Sequences – /api/sequences

Saved sequences of the admin UI, run through the API ([07](07-operations.md#saved-sequences-through-the-api)). Any valid token may call them; each step then needs the caller's own role for that step.

| Method | Path | Roles | PowerShell |
|---|---|---|---|
| GET | `/api/sequences` | any valid token | `Get-ApiSequences` |
| POST | `/api/sequences/{sequenceName}/run` | any role of this API + the role of every step; `ConfirmComputerName` for irreversible steps | `Invoke-ApiSequence` |

## Admin – /api/admin (used by the admin UI)

| Method | Path | Roles |
|---|---|---|
| GET | `/api/admin/ui-config` | none – sign-in settings for the UI |
| GET | `/api/admin/me` | any valid token |
| GET | `/api/admin/status` | Monitoring.Read or Admin.Manage |
| POST | `/api/admin/health/refresh` | Monitoring.Read or Admin.Manage |
| GET | `/api/admin/routes` | Monitoring.Read or Admin.Manage |
| GET | `/api/admin/audit?limit=&minStatus=&search=` | Monitoring.Read or Admin.Manage |
| GET | `/api/admin/settings` | Monitoring.Read or Admin.Manage |
| PUT | `/api/admin/settings` | Admin.Manage |
| GET | `/api/admin/configuration` | Admin.Manage – `appsettings.json` with defaults, editable keys, keys waiting for a restart, the default of each key (`defaults`) |
| PUT | `/api/admin/configuration` | Admin.Manage – body `{"expiry.warnDays": 45}`, `null` resets a key to its default; only the keys of [06](06-configuration.md#service-configuration-in-the-admin-ui), others 400; applies after a restart |
| POST | `/api/admin/restart` | Admin.Manage – 202, the Windows service restarts (event 1005); 503 when not running as a service |
| POST | `/api/admin/expiry/refresh` | Monitoring.Read or Admin.Manage |
| GET | `/api/admin/sequences` | any role of this API |
| PUT | `/api/admin/sequences` | Admin.Manage |
