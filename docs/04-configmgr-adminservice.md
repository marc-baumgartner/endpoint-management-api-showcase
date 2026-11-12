# 04 – ConfigMgr and the AdminService

The service talks to ConfigMgr through the **AdminService**, the REST API that ConfigMgr itself provides on the SMS Provider ([Microsoft docs](https://learn.microsoft.com/configmgr/develop/adminservice/usage)). It exposes the site's WMI classes over HTTPS, so no SDK and no DCOM connection to the site server are needed.

| AdminService route | Use |
|---|---|
| `GET /AdminService/wmi/<Class>?$filter=…&$select=…` | queries on site WMI classes (`SMS_R_System`, `SMS_Collection`, …) |
| `POST /AdminService/wmi/<Class>.<Method>` | static WMI methods (`SMS_ClientOperation.InitiateClientOperation`, …) |
| `POST/PATCH/DELETE /AdminService/wmi/<Class>(<key>)` | create, change and delete instances |

## Authentication: Windows sign-in as the service account

The AdminService accepts Windows authentication only, and it requires **Extended Protection** (channel binding): the Kerberos or NTLM sign-in must be bound to the TLS connection. Node.js cannot do that. On Windows the service therefore hands AdminService requests to a small Windows PowerShell process (`api\windows\AdminServiceProxy.ps1`), which it starts itself and keeps running. That process signs in as the service account through the Windows HTTP stack, exactly like `Invoke-RestMethod -UseDefaultCredentials`. Nothing extra needs to be installed.

If the AdminService answers **401** (sign-in rejected), the service drops the Kerberos tickets of the service account (`klist purge`), starts a new helper process and sends the request once more – at most once a minute. This covers a fresh service start after an install, and Kerberos tickets that the AdminService stops accepting after some hours. `app.log` shows "AdminService answered 401 … request sent again" and whether the retry was accepted; once an hour it also writes the Kerberos state before the recovery (`klist tgt`, `klist`), to find out why the tickets were rejected. If it still answers 401:

- **The AdminService accepted the service before** (e.g. it worked for hours, then every lookup shows *Unavailable*): the logon session of the service went stale, and only a new one helps – the same as `Restart-Service` by hand. The service writes the Kerberos state (`klist tgt`, `klist`) to `app.log`, event **1004** to the Event Log and restarts itself; WinSW starts it again within 10 seconds. Turn this off with `configMgr.restartOnKerberosFailure = false`.
- **It never accepted it since the start:** the account really is not accepted – no restart. Check that it is an administrative user in ConfigMgr and that `klist get HTTP/<providerHost>` works as that account.

`configMgr.transport` selects the way: `windows` (default on Windows) or `node` (Negotiate from Node.js with the optional `kerberos` module – only for AdminService installations without Extended Protection). The release compiles `kerberos` from source; if the build machine had no compiler it is missing and `node` answers 500 (*The "kerberos" package is not installed …*), so check `api\node_modules\kerberos` before you switch.

Quick check on the API server, as the service account:

```powershell
Invoke-RestMethod 'https://<provider FQDN>/AdminService/wmi/SMS_Site?$select=SiteCode' -UseDefaultCredentials
```

If that returns the site code, the service can sign in too.

1. Create the gMSA and allow the API server to use it:
   ```powershell
   # once per domain, on a DC: the key the gMSA passwords are derived from
   Get-KdsRootKey                                   # nothing? then:
   Add-KdsRootKey -EffectiveImmediately             # usable after ~10 h; in a lab: -EffectiveTime ((Get-Date).AddHours(-10))

   # the gMSA; the API server's computer account may retrieve its password
   New-ADServiceAccount gmsa-emapi -DNSHostName gmsa-emapi.contoso.local -PrincipalsAllowedToRetrieveManagedPassword 'SCCM01$'

   # on the API server (module: Install-WindowsFeature RSAT-AD-PowerShell)
   Install-ADServiceAccount gmsa-emapi
   Test-ADServiceAccount gmsa-emapi                 # must return True
   ```
   `SCCM01$` is the computer name of the API server with `$`. "Cannot find an object with identity" means the gMSA was not created yet. `False` from `Test-ADServiceAccount`: the server does not know its new right yet – `klist purge -li 0x3e7` or a restart, then test again.
2. ConfigMgr console → *Administration → Security → Administrative Users → Add* `CONTOSO\gmsa-emapi$` with a role that covers what the API does (devices, collections, user device affinity, client notification, status messages). Limit the scope to the collections the API manages.
   - The account picker often does not list gMSAs: type the name by hand, with the `$` at the end.
   - Easier to keep: create a group such as `CONTOSO\G_EMAPI_ConfigMgr`, put the gMSA into it and add the **group** as administrative user. The picker always finds groups, and the account can change later without touching ConfigMgr. After adding the gMSA to the group, restart the service: the membership arrives with a new Kerberos ticket.
   - A normal domain account (`CONTOSO\svc-emapi`) works the same way; it is enough for a lab, but someone knows its password and has to change it.
3. `configMgr.providerHost` = FQDN of the SMS Provider exactly as in its certificate. The AdminService uses the site's certificate (PKI or the ConfigMgr-generated one). With a self-signed certificate set `trustedCertificateThumbprint` to its thumbprint (only that certificate is accepted); `allowUntrustedCertificate: true` accepts any certificate and belongs in a lab at most.

## Windows helper process

The same PowerShell script (`api\windows\AdminServiceProxy.ps1`) also runs the two ConfigMgr operations that only exist on Windows, so the service needs no .NET component of its own.

The script handles one request at a time. So that one caller cannot hold up the others, the service runs it twice: one process for the AdminService, event log, LAPS and DDR, and one for the policy reset, which waits for the client over DCOM and can take the full `requestTimeoutMs` for an offline device. Every request carries its deadline (`requestTimeoutMs` + 5 s): a request still waiting in the queue when its caller already got 504 is skipped instead of being run later. The service starts Windows PowerShell by its full path (`%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe`).

| Operation | Used by | What it does | Needs |
|---|---|---|---|
| `resetPolicy` | `POST …/policy-reset` with `FullReset: true` | `root\ccm SMS_Client.ResetPolicy(1)` on the client over WMI/DCOM. Only for a device that ConfigMgr knows as an active client (`SMS_R_System.Client = 1`, otherwise 409) and whose name is a plain computer name (no IP address, no `:`, `\` or `/`, otherwise 400), so the service account does not sign in to any host a caller names | WMI/DCOM from the API server to the client; the service account is local admin on the client |
| `ddr` | `POST /api/configmgr/devices` (RegisterDevice) | discovery data record with SID, OUs and groups, written to the site server inbox | `configMgr.messagingSdkPath` = local path of `Microsoft.ConfigurationManagement.Messaging.dll` (loaded only with a valid Microsoft signature; if Windows cannot check the signature for the service account, only when no ordinary user may change the file or its folder); write access of the service account to `\\<siteServer>\SMS_<site>\inboxes\ddm.box` |

### Required for Register Device (DDR)

Only needed if you use *Register Device* (`POST /api/configmgr/devices`) and want OU and groups in ConfigMgr at once, instead of after the next AD System Discovery. Without it the device is still imported.

**How it works:** after the import the service reads the computer from AD (SID, OUs, groups) and the Windows helper writes a discovery data record (DDR) into the inbox `ddm.box` of the site server. The Messaging SDK first tries to find the inbox in the site server's registry (`HKLM\SOFTWARE\Microsoft\SMS`). A service account normally may not read that key ("Requested registry access is not allowed"); the helper then writes the file straight into `\\<siteServer>\SMS_<site code>\inboxes\ddm.box`. **No registry right is needed.** No new share is created either: `SMS_<site code>` is the share ConfigMgr creates on the site server; it points to the installation folder, e.g. `E:\Microsoft Configuration Manager`.

**Setup, once (on the site server, as administrator; example site code `LAB`, service account `CONTOSO\gmsa-emapi$`):**

1. Share permission **Change** on `SMS_LAB` (check first; not needed if a group of the account already has Change):

   ```powershell
   Get-SmbShareAccess SMS_LAB
   Grant-SmbShareAccess SMS_LAB -AccountName 'CONTOSO\gmsa-emapi$' -AccessRight Change -Force
   ```

2. NTFS **Modify** on the inbox folder:

   ```powershell
   icacls 'E:\Microsoft Configuration Manager\inboxes\ddm.box' /grant 'CONTOSO\gmsa-emapi$:(OI)(CI)M'
   ```

3. In `appsettings.json` on the API server, the local path of `Microsoft.ConfigurationManagement.Messaging.dll` from the **console** (`<console>\bin\`; on a site server `<installation folder>\AdminConsole\bin\`). Not the copy below `bin\X64\CMRestProvider`: that one belongs to the AdminService itself.

   ```json
   "messagingSdkPath": "E:\\Microsoft Configuration Manager\\AdminConsole\\bin\\Microsoft.ConfigurationManagement.Messaging.dll",
   "siteCode": "LAB"
   ```

   The helper loads it only from a local path and only with a valid Microsoft signature; if Windows cannot check the signature for the service account, only when no ordinary user may change the file or its folder (the case for the console folder). `configMgr.siteServer` is needed only when the site server is not `configMgr.providerHost`.

4. Restart the service: `Restart-Service EndpointManagementAPI`.

**Test:** the computer needs an AD account – without one there is no SID and the service skips the DDR ("has no AD object (or no SID); DDR skipped.").

```powershell
New-AdComputerAccount -ComputerName 'DDRTEST01' -TargetOU 'OU=Clients,DC=contoso,DC=local'
Register-ConfigMgrDevice -ComputerName 'DDRTEST01' -BiosGuid ([guid]::NewGuid().ToString())
```

`app.log` must show `ConfigMgr: DDR submitted for "DDRTEST01" (ResourceID …)`. After a minute or two the device shows *System OU Name* and *Security Group Name* in the console. Remove the test objects afterwards (ConfigMgr and AD).

| `app.log` says | Cause | Fix |
|---|---|---|
| `DDR skipped` (no AD object or no SID) | the computer has no account in AD yet | create it first (prestage), then register |
| `configMgr.messagingSdkPath is not set` | step 3 missing | set the path, restart |
| `ConfigMgr Messaging SDK not found` / `no valid Microsoft signature` / `may change …` | wrong path, wrong file, or a folder ordinary users may write to | path of the console DLL (step 3) |
| `Writing the DDR to \\…\ddm.box failed: Access is denied` | share or NTFS right missing | steps 1 and 2 |
| `Writing the DDR to \\…\ddm.box failed: … network path was not found` | wrong site server or site code | `configMgr.siteServer`, `configMgr.siteCode` |

## Operations

"Lab check" marks calls whose request shape follows the Microsoft documentation and the AdminService conventions but that have not yet run against a lab site – run each once before production (checklist below).

| Function | AdminService call | Status |
|---|---|---|
| Device exists, resource ID | `GET wmi/SMS_R_System?$filter=Name eq '…'` | tested in the lab |
| Primary users (read) | `SMS_UserMachineRelationship`, `Sources` contains 2 (administrator defined) | tested in the lab |
| Add primary user | `SMS_UserMachineRelationship.CreateRelationship` (`MachineResourceId`, `UserAccountName`, `SourceId 2`, `TypeId 1`); user from `SMS_R_User.UniqueUserName` | lab check |
| Remove primary user | `DELETE wmi/SMS_UserMachineRelationship(<RelationshipResourceID>)` | lab check |
| Sync primary users | add/remove the difference; users that cannot be added go to `configMgr.pendingPrimaryUserDirectory`; the list is stored in the device variable `configMgr.primaryUserVariable` | lab check |
| Policy refresh | client notification `SMS_ClientOperation.InitiateClientOperation` type **8** | tested in the lab |
| Hardware inventory | client notification type **12** | lab check |
| Policy reset (`FullReset: false`) | client notification type 8 (download and evaluate) | lab check |
| Policy reset (`FullReset: true`) | `SMS_Client.ResetPolicy(1)` on the client through the Windows helper | lab check |
| Status messages | `SMS_StatusMessage.Raise*StatusMsg` (`MessageText`, `MessageType`) over **WMI through the Windows helper**, not the AdminService: the provider wants the WMI context values `ApplicationName`, `MachineName` and `LocaleID`, which the AdminService cannot pass ("ApplicationName is not set in the context"). The message gets the target computer as System and the component as Component, so it shows under the device (Monitoring → Status Message Queries → All Status Messages from a Specific System, time range wide enough) with the description as text. It does not show under *Show Status Messages* on the device itself: that view lists the messages of the ConfigMgr client, and these arrive through the SMS Provider (ModuleName *SMS Provider*, type *Detail*). If the SMS Provider runs on another server than the API, the service account needs remote WMI (DCOM) access to it. Components starting with `SMS_` or `CCM` (ConfigMgr's own) are refused, so a message cannot pose as one of them | lab check |
| Collection: member?, add, remove | `SMS_FullCollectionMembership`; `POST wmi/SMS_Collection('<id>')/AdminService.AddMembershipRule` / `DeleteMembershipRule` with `#AdminService.SMS_CollectionRuleDirect`; waits for `CurrentStatus` 1 or 5 | lab check |
| Refresh collection | `POST wmi/SMS_Collection('<id>')/AdminService.RequestRefresh` | lab check |
| OSD collection, restaging | `configMgr.osDeployments` → direct rule (+ PXE flag) | lab check |
| Remove from OSD collection | `SMS_Advertisement` → collection → delete rule | lab check |
| Clear PXE flag | `SMS_Collection.ClearLastNBSAdvForMachines` | lab check |
| Device variables (set, remove, replace all) | read `wmi/SMS_MachineSettings(<id>)`, write the whole instance with `POST wmi/SMS_MachineSettings` | lab check |
| Register device by BIOS GUID | `SMS_Site.ImportMachineEntry` | lab check |
| DDR after registration | Windows helper with the ConfigMgr Messaging SDK | lab check |
| Delete device (name / BIOS GUID) | `DELETE wmi/SMS_R_System(<id>)` | lab check |
| USMT association add / remove / status | `SMS_StateMigration.AddAssociationEx` (`MigrationBehavior 0`: capture and restore all user accounts, like the console) / `DeleteAssociation`, query on `SMS_StateMigration` | lab check |
| Application deployments (device, user) | `SMS_FullCollectionMembership`, `SMS_DeploymentInfo`, `SMS_Application`, `SMS_ApplicationAssignment`, collection query rules | lab check |

The AdminService rejects a method parameter whose name is not exactly the documented one – a different case is enough (`MachineResourceID` instead of `MachineResourceId`). The parameter names of every WMI method the service calls are listed in `api/src/connectors/wmiMethods.ts`, taken from the Configuration Manager SDK reference; each call is checked against that list in production and in every test.

### Behaviour

| Situation | Answer |
|---|---|
| Unknown device, collection, user or advertisement | 404 with a message, e.g. `Device 'PC01' not found in ConfigMgr.` (`…/exists` and `…/membership` answer `false`) |
| Device already registered (same BIOS GUID) | 409 |
| Collection still refreshing after `collectionPollTimeoutSeconds` | 409 `Collection '…' is not ready …` |
| Device with a temporary resource ID (2…) in SyncPrimaryUsers | 409 |
| Membership already as requested (add twice, remove a non-member) | 200, nothing changed |
| Full policy reset or DDR without the Windows helper | 503 |

### Lab checklist

With a test device (e.g. `DEV-ENV1`), from the PowerShell module or the admin UI: RegisterDevice (new BIOS GUID) → DeviceExists → add/remove/sync primary user → add/remove collection (direct rule visible in the console) → RefreshCollection → set/remove/replace device variables (console: device → Properties → Variables) → ClearPxeFlag → OSD collection add/remove → PolicyRefresh, HardwareInventory, PolicyReset (both) → StatusMessage (console: Status Message Queries) → USMT add/status/remove → applications (device, user) → DeleteDevice.

### Rights of the service account in ConfigMgr

The account needs a security role that allows: read devices and users, *Modify Resource* and *Delete Resource* on devices, *Modify* on the collections it manages (direct rules, refresh), *Notify Resource* (client notification), *Manage OSD and ISV Proxy Data* (USMT associations), import computer entries, and status message creation. The built-in *Full Administrator* covers everything; for production create a custom role and limit the scope to the managed collections.
