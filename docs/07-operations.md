# 07 – Operations

## Admin UI

The UI is in English and follows Apple's Human Interface Guidelines for macOS (system font, grouped lists, sidebar, pull-down menus, sheets and alerts, light and dark appearance), built with Base UI components. On narrow screens the sidebar becomes a tab bar.

`https://<host>/<path>/` – sign in with your work account. What you see depends on your roles ([03](03-entra-setup.md)).

| Page | Shows | Role |
|---|---|---|
| Status | API version with the build commit (e.g. `2.0.0 (bde56fd)`) and uptime, each connector with state and latency, certificates and secrets that expire soon, traffic of the last hour, areas on/off. An area switched off in Settings shows at once: ConfigMgr or AD as *Off, disabled by an administrator*; for Microsoft Graph *Off* only when Intune and Entra ID are both off, otherwise *Connected* with the note *Intune disabled by an administrator* (or Entra ID) | Monitoring.Read or Admin.Manage |
| Devices | device record: one computer name or serial number looked up in ConfigMgr, AD, Intune and Entra ID at once, with the actions of each system (policy refresh, collections, groups, OU move, sync, wipe, …) and sequences of actions | the Read roles of each system; ReadWrite for the actions |
| Audit | the last calls since start, filter by text or errors | Monitoring.Read or Admin.Manage |
| Routes | all 91 routes with operation name and the roles they need | Monitoring.Read or Admin.Manage |
| Settings | integrations on/off; sections General (rate limits), Logging (log level), ConfigMgr, Active Directory, Entra ID and Intune, Certificate Expiry with the values of `appsettings.json`, a few changeable, restart of the service; notice ([06](06-configuration.md#service-configuration-in-the-admin-ui)) | read: Monitoring.Read (values of `appsettings.json`: Admin.Manage); change: Admin.Manage |

A serial number of an enrolled device opens that device ("Found by serial number …"). A serial number that is only registered in Windows Autopilot shows the Autopilot record under Intune ("Autopilot Only") with the Autopilot actions: set the group tag, delete the record. When AD has no computer account with the name, the AD menu offers *Create Computer Account* (prestage in an OU).

The device record calls the same `/api/...` routes as every other client, with the signed-in person's token – it can do nothing a person could not do with their roles anyway.

The *Actions* menu of a system lists its actions in sections (for Intune: Device, Security, Assignment), destructive ones last. Actions the person lacks the role for stay in the menu, greyed out, with the role they need. Actions that cannot run on this device stay visible too, greyed out with the reason: *No computer account*, *Not in ConfigMgr*, *Not enrolled* (only registered in Autopilot), *Already exists* (Create Computer Account), *None stored* (LAPS, BitLocker), *Unavailable* (system not reachable). LAPS passwords and BitLocker recovery keys open a sheet that shows the value only on request, with a copy button; every lookup is a critical action in the audit and event log. Values an action changes (extension attribute, group tag, description, enabled) show at once, even while Entra ID or Intune still report the old value. Harmless, repeatable actions (sync, policy refresh) run at once and confirm below the system; actions that need input open a sheet; restart and lock ask in an alert; irreversible actions (delete, wipe, retire) ask for the computer name before they run. The catalogue of all device actions is `apps/admin-ui/src/deviceActions.ts`; the API uses the same list to run saved sequences (`npx tsx scripts/gen-action-catalogue.ts` after changing it).

### Links and navigation

The address names the page and the device: `…/#devices/CH-NB-04711`. Reload (F5), back and forward keep the device, and the link can be sent to a colleague; they see it with their own roles. A serial number is replaced by the device name it belongs to. The logo and name in the sidebar lead home: Status, or Devices for people without Monitoring.Read. The browser tab shows the page or the device name.

- **Recent:** below the search, the last five devices found, kept in this browser only, per person, and removed at *Sign Out* (so the next person on a shared browser does not see them).
- **Copy:** the icon next to the device name and the serial number copies it.
- **Audit link:** *Calls for <device> in the audit log* opens Audit filtered to that device (needs Monitoring.Read). The search and the *Errors* filter of Audit are part of the address too (`#audit?q=PC01&show=issues`).
- **Messages at the bottom (toasts)** that do not move the page: a setting changed, a sequence saved or deleted, something copied (a copied LAPS password or BitLocker key is removed from the clipboard after 30 s), *Can’t reach the API. Retrying…* while the service does not answer (gone with the next answer), *Your sign-in has expired* with *Sign In Again*. Results of a device action stay next to the system, errors in a form next to the field.
- **New version:** a tab keeps running the admin UI it loaded, also after an update of the service. Once a minute while the tab is visible (and when it becomes visible again) it checks whether the server has a newer admin UI; if so, the message *A new version of Endpoint Management is installed. Reload to use it.* stays at the bottom with *Reload*. The version in the account menu is the service's; it can be newer than the screens of a tab that was not reloaded.
- **Phone:** below 768 px a tab bar replaces the sidebar, and Audit and Routes show each call or route as a list row (time, status and duration, then caller and request) instead of a wide table.
- **Unsaved sequence:** closing the sequence sheet with steps that are not saved asks *Discard them?*; leaving or reloading the page asks the browser's own question.

### Account menu

The picture is the person's profile photo from Entra ID (read by the API with `User.Read.All`, which it has anyway); without a photo the initials show.

The button with the person's initials at the top right opens the account menu:

- name, sign-in name, environment and version
- *Appearance*: Light, Dark or Auto (follows the operating system), kept per browser
- *Roles and Session*: when the person signed in, until when the current token is valid, the tenant, and every app role by area, the assigned ones ticked
- *API Documentation* (`/api/docs`) and *Sign Out*

## Sequences

*Devices → look up a computer → Run Sequence* runs several device actions one after another, like building blocks. Example *Hand Over to User*:

1. Sync (Intune)
2. Wait 30 seconds
3. Set Primary User (Intune) – user name asked for when the sequence runs
4. Add Primary User (ConfigMgr)
5. Request Policy Refresh (ConfigMgr)

- *Add Step* offers every action the person has the role for, plus *Wait* (1–600 s). Actions that cannot run on the device are greyed out with the reason, the same as in the Actions menu; a step after *Create Computer Account* counts the account as there, a step after a delete (*Delete from ConfigMgr*, *Delete Computer Account*, *Delete from Intune*) the record as gone. A loaded saved sequence with such a step shows why and does not run. Steps can be moved and removed; up to 20 per sequence.
- Each step is a normal API call with the signed-in person's token: roles, rate limits, audit log and event log apply per step, exactly as if the actions were clicked one by one.
- *Stop at the first failed step* (default) skips the rest after an error; without it, the remaining steps still run. Each step shows done, failed (with the message) or skipped.
- If a step is irreversible (delete, wipe, retire), the computer name has to be typed before *Run*.
- People with `Admin.Manage` save a sequence for everyone (*Save as Sequence*, stored in `config\sequences.json`). Fields left empty in a saved sequence are asked for each time – typically the user name. Everyone else can load saved sequences and build their own ad hoc.
- **Change a saved sequence:** pick it under *Saved Sequences*, change the steps, *Save* (the name is filled in; the same name overwrites it). While the steps differ from the saved ones, *Changes not saved* shows next to *Save*.
- **Delete a saved sequence:** pick it, *Delete*, confirm *Delete “…” for everyone?*.
- Without `Admin.Manage` there is no *Save* and *Delete*; the sheet says so, and those people can still load and run saved sequences.
- In the admin UI, sequences run in the browser; closing the window stops the remaining steps.

### Saved sequences through the API

A saved sequence can also run without the UI – from a task sequence, ServiceNow or a script. The service runs it the same way: one API call per step, with the **caller's** token, so each step needs the caller's own role and appears in the audit log.

```powershell
(Get-ApiSequences).data | Select-Object name, @{ n = 'needs'; e = { $_.steps.needs | Sort-Object -Unique } }
Invoke-ApiSequence -Name 'Hand Over to User' -ComputerName 'CH-NB-04711' -Values @{ userName = 'amuster'; '3.userName' = 'anna.muster@contoso.ch' }
```

```http
POST /api/sequences/hand-over-to-user/run
{ "ComputerName": "CH-NB-04711", "Values": { "userName": "amuster", "3.userName": "anna.muster@contoso.ch" } }
```

- The sequence is found by id or name (case is ignored). Names are unique, and no name may equal another sequence's id; saving such a list is refused.
- Listing needs any app role of this API. Running needs the role of every step: all steps are checked before the first one runs, and a missing role gives 403 with the steps it concerns, e.g. `step 4 (Add Primary User) needs ConfigMgr.ReadWrite`. Each step is checked again when it runs.
- A sequence with an irreversible step (delete device or computer, wipe, retire, delete Autopilot record) needs the computer name a second time, as the admin UI asks for it: `"ConfirmComputerName": "CH-NB-04711"` (`Invoke-ApiSequence -ConfirmComputerName`). Without it: 400, nothing runs.
- `Values` fills the fields left empty in the saved sequence: `userName` fills every empty `userName`, `3.userName` sets the field of step 3 only (steps counted from 1, also overrides a saved value). Values that are still missing are refused with 400 before any step runs; `GET /api/sequences` lists them per step under `needs`.
- `StopOnError` (optional) overrides the setting saved with the sequence.
- Answer: 200 with every step `done`; otherwise the status of the step that failed while running (e.g. 404, 409) with `error` = `Step 4 (Add Primary User) failed: …` and the list of steps (`done`, `failed`, `skipped`) in `data`.
- The call returns when the last step is done, waits included. Waits may add up to 10 minutes per run; `Configure-IISProxy.ps1` sets the IIS proxy timeout to 15 minutes, `Invoke-ApiSequence` waits 15 minutes by default (`-TimeoutSec`).

## Windows Event Log

The service writes to its own log **Applications and Services Logs → Endpoint Management API** (source `EndpointManagementAPI.Service`). The installer creates the log (20 MB, overwrite as needed) and allows the service account to write to it. Every event ID has one meaning, so you can filter, forward (Windows Event Forwarding, Splunk, Sentinel) or attach a scheduled task to an ID.

| ID | Level | Meaning |
|---|---|---|
| 1000 | Information | service started (version, environment, address, areas switched off) |
| 1001 | Information | service stopping |
| 1002 | Warning | configuration warning, e.g. admin UI without `entra.adminUiClientId`, `appsettings.json` changed but invalid, changed sections that need a restart, or a damaged `sequences.json` (kept as `sequences.json.damaged-<time>`) |
| 1003 | Error | service could not start (e.g. port in use) |
| 1004 | Warning | service restarts itself: the AdminService keeps rejecting the Kerberos sign-in it accepted before ([04](04-configmgr-adminservice.md)) |
| 1005 | Warning | service restart requested in the admin UI (*Settings*, *Restart Service…*), with the person |
| 2010 / 2011 | Error / Information | ConfigMgr AdminService not responding / responding again |
| 2020 / 2021 | Error / Information | Active Directory (LDAP) not responding / responding again |
| 2030 / 2031 | Error / Information | Microsoft Graph not responding / responding again |
| 3000 | Information | change made (POST, PUT, DELETE): caller, route, computer, status |
| 3001 | Error | change failed in the target system (5xx) |
| 3100 | Warning | critical action: delete device, computer or Autopilot record, wipe, retire, BitLocker recovery key or LAPS password read |
| 3101 | Error | critical action failed |
| 4000 | Warning | access denied (403) – once per caller and route every 10 minutes |
| 4001 | Warning | rate limit reached (429) – once per caller every 10 minutes |
| 4100 | Warning | settings changed in the admin UI (who, which settings) or in appsettings.json (which settings) |
| 4200 | Information | plain values in `secrets.json` encrypted with DPAPI |
| 5000 | Warning | certificate or secret expires within `expiry.warnDays` – once per item and day |
| 5001 | Error | certificate or secret has expired – once per item and day |

Reads (GET) are not written to the event log – they are in `audit.log`. The message text is the first insertion string of the record:

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'Endpoint Management API'; Id = 3100, 5000, 5001 } -MaxEvents 20 |
    Format-Table TimeCreated, Id, LevelDisplayName, @{ n = 'Message'; e = { $_.Properties[0].Value } } -Wrap
```

If the log is missing (service installed by hand), create it once as administrator:

```powershell
New-EventLog -LogName 'Endpoint Management API' -Source 'EndpointManagementAPI.Service'
```

## Expiry warnings

Every `expiry.intervalHours` (default 6 h) and on *Check Now* in the UI, the service checks what expires:

| Source | What | Needs |
|---|---|---|
| This API | the HTTPS certificate clients see, read over the host names they use (e.g. `endpoint-management.contoso.local`). Taken from `server.publicUrls` when set; otherwise learned from successful HTTPS requests of administrators (`Admin.Manage` or `Monitoring.Read`) – the first sign-in to the admin UI after an install is enough – and a new host name is checked at once. Only DNS names on the `expiry.iisPorts` count, and a learned name is forgotten after 30 days without a request. A certificate whose expiry date cannot be read is listed as a problem of that source | – |
| IIS | HTTPS certificates of the http.sys bindings on this server on the ports in `expiry.iisPorts` (default 443 – the API's own binding) | Windows |
| ConfigMgr | server certificate of the AdminService (`configMgr.providerHost:443`) | ConfigMgr enabled |
| Active Directory | server certificate of every `ldaps://` domain controller | AD enabled |
| Graph certificate | the PEM at `entra.graphCertificatePath` | `graphAuth = certificate` |
| App registrations | client secrets and certificates of `EndpointManagement.Service` and of every client registration with app roles on it (TaskSequence, ServiceNow, …) | Graph permission `Application.Read.All` |

The same certificate seen in several places – e.g. the IIS binding on 443 and the AdminService on the same server – is one row; the second line names the other places ("also used by …"). Status lists every certificate and secret found, soonest first, and marks what expires within `expiry.warnDays` (default 30) or has expired, with owner and date. Sources that cannot be checked (e.g. permission missing) are listed with the reason. The event log gets 5000 / 5001 once per item and day.

## Rate limits

Per caller – an app registration or a person – in a sliding window:

| Limit | Default | Counts |
|---|---|---|
| reads | 300 per minute | GET |
| changes | 60 per minute | POST, PUT, DELETE |
| critical actions | 20 per hour | delete device, computer or Autopilot record, wipe, retire, BitLocker recovery key, LAPS password |
| ConfigMgr requests at the same time | 6 per caller | requests to `/api/configmgr/…` that are still running (fixed; they share the Windows helper with every other caller) |

Above the limit the API answers **429** with a `Retry-After` header and the message "Too many requests: the limit is … Try again in … s." (event 4001). The PowerShell module shows that message. The values are those in `appsettings.json` (`rateLimit`): people with `Admin.Manage` change them under *Settings → General*, which writes them into the file, and a change made in the file shows in the UI – both without a restart ([06](06-configuration.md#log-level-and-rate-limits-one-source-ui-and-file-in-step)). The admin UI's own calls (`/api/admin/…`) are not limited; the device actions it runs are, like every other client. Give a monitoring probe its own app registration so its health checks do not use up a client's reads.

## Logs

| File | Content |
|---|---|
| `logs\app.log` | application log, JSON lines (pino), level from settings |
| `logs\audit.log` | one JSON line per API call: time, request ID, method, path, route, status, duration, caller kind, caller, client ID, IP, error. Not recorded: successful reads the admin UI makes for itself (`GET /api/admin/*` such as status, settings, audit, routes, account, and `GET /api/sequences`); they repeat while a page is open. Their failures (401, 403, 5xx) are recorded, and so is every call on a device, including reads |
| `logs\service\*.log` | stdout/stderr of the process (WinSW), rolled at 10 MB, 8 files |

`app.log` and `audit.log` are not rotated by the service. Rotate them with your log shipper or a scheduled task, or forward them (Splunk, Elastic, Sentinel) – the JSON format needs no parsing rules.

Read a log on the server:

```powershell
Get-Content C:\ProgramData\EndpointManagementAPI\logs\audit.log -Tail 20 | ConvertFrom-Json | Format-Table time, status, method, path, caller
```

## Health

`GET /api/monitoring/health` (role `Monitoring.Read`) answers **200** when every enabled connector works, **503** otherwise – usable for load balancer and monitoring probes:

```json
{ "status": "Degraded", "environment": "Production",
  "checks": { "configmgr": "OK", "active_directory": "OK", "graph": "Degraded: AADSTS7000222: …" } }
```

Results are cached for 30 s; *Check Now* in the UI forces a new check.

## Lab test

`deploy\windows\Test-EndpointManagementApi.ps1` (also in the install folder) runs the routes against a test device with a client app registration and prints OK / FAIL per call:

```powershell
& 'C:\Program Files\EndpointManagementAPI\deploy\windows\Test-EndpointManagementApi.ps1' `
    -BaseUrl https://<host>/<path> -TenantId <tenant> -ClientId <client> -ApiAppId <Service app id> `
    -ComputerName DEV-ENV1 -UserName <test user> -Collection '<test collection>' -AdGroup <test AD group> -EntraGroup <test Entra group> -Write
```

The script asks for the client secret with hidden input, so it does not end up in the command history. The test groups must be allowed by the scope settings (`managedGroups`) and must not be protected (`G_EMAPI_*`), see [06](06-configuration.md#scope-what-a-readwrite-role-may-reach). Without `-Write` only reading calls run. `-Write` makes changes and reverts them at once; `-Destructive` also registers and deletes a throw-away device. Results are saved as CSV in `%TEMP%`.

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| 500 "Internal error. Request ID …" | an error the service did not expect (programming error, file system, network) | the details are in `app.log` and in the audit log under that request ID; target system errors (Graph, AdminService, LDAP) keep their own message |
| Every call 401 "Token validation failed." | `app.log` "token refused": `unexpected "aud"` = the client requests a token for another API; `not accepted for people` = a person signs in through another app than the admin UI; `no client is allowed for people` = neither `entra.adminUiClientId` nor `entra.allowedUserClients` is set; `without a delegated scope` = a token of a person without `scp` | scope must be `api://<apiAppId>/.default` ([10](10-clients.md)); `entra.allowedUserClients` ([06](06-configuration.md)) |
| 403 "Required role(s): …" | role not granted to the client or person | Enterprise app → Users and groups / client API permissions + admin consent |
| 403 "HTTPS is required." | IIS does not set `X-Forwarded-Proto`, the request came over plain HTTP, or IIS runs on another server than the one in `server.trustProxy` | allowed server variables on the IIS site (`Configure-IISProxy.ps1`); use `https://`; add the proxy address to `server.trustProxy` |
| 403.4 from IIS | plain HTTP to the site or application ("Require SSL" is set) | use `https://` |
| Installer: "Files in the release folder … can be changed by …" | the release was unpacked into a folder whose files other users may change (a share, `C:\Users\Public`) | unpack into a folder below `C:\Temp` or another local folder ([02](02-installation.md#install)) |
| Installer: "… does not match the release manifest" | a file was changed, added or is missing after the build | download and unpack the release again into an empty folder |
| ConfigMgr "Degraded: AdminService returned 401" | Kerberos failed | service runs as the gMSA? SPN `HTTP/<provider>` exists? gMSA is an administrative user in ConfigMgr? |
| "unable to verify the first certificate" | internal CA not trusted | service must start Node with `--use-system-ca` (`EndpointManagementAPI.xml`), and the CA must be in the server's *Trusted Root Certification Authorities*, see [05](05-active-directory.md) |
| AD "Degraded: Invalid credentials" | bind password wrong or expired | `secrets.json` / `EMAPI_LDAP_BIND_PASSWORD` |
| 404 "Device '…' not found in ConfigMgr." (or AD, Intune, Entra ID) | the object does not exist | check the name; `…/exists` routes answer `false` instead |
| 409 "Collection '…' is not ready" | collection refresh still running after `collectionPollTimeoutSeconds` | retry later |
| 409 on a full policy reset "… has no ConfigMgr client" | the device is no active ConfigMgr client, so the service does not connect to it | install or repair the client; a normal policy reset (`FullReset: false`) still works |
| 403 "… is nested in the protected group …" | the AD group is a member (on any level) of a protected group | change the nesting, or use a group outside it ([06](06-configuration.md#scope-what-a-readwrite-role-may-reach)) |
| 503 "… needs the Windows helper" | full policy reset or DDR on a service not running on Windows / `transport: node` | run the service on Windows with `configMgr.transport = windows` |
| RegisterDevice works, log "DDR … failed" or "DDR skipped" | no AD account yet, `configMgr.messagingSdkPath` missing, or no write access to `ddm.box` | [04 – Required for Register Device (DDR)](04-configmgr-adminservice.md#required-for-register-device-ddr): setup, test and the messages |
| 503 "… is disabled by an administrator" | area switched off | admin UI → Settings |
| Service stops right after start | invalid configuration | `logs\service\*.err.log` names the wrong key |
