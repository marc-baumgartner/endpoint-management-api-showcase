# 06 – Configuration

Three files in `C:\ProgramData\EndpointManagementAPI\config` (folder set by `EMAPI_CONFIG_DIR`):

| File | Content | Changed by | Restart needed |
|---|---|---|---|
| `appsettings.json` | all non-secret settings | administrators, by hand; log level, rate limits and the keys listed under [Service configuration in the admin UI](#service-configuration-in-the-admin-ui) also in the admin UI | yes – except `logging.level` and `rateLimit` |
| `secrets.json` | passwords and keys | administrators, by hand | yes |
| `runtime.json` | areas switched off, notice | admin UI (*Settings*) | no |
| `public-hosts.json` | host names clients use over HTTPS, for the certificate check | the service, from successful signed-in requests | no |

The service validates `appsettings.json` at start and refuses to run with a clear message if a value is wrong.

### Log level and rate limits: one source, UI and file in step

`logging.level` and `rateLimit` are read from `appsettings.json` and nowhere else. The admin UI shows exactly these values:

- **Changed in the UI** (*Settings*): written into `appsettings.json` at once and in effect at once. Event 4100 names the person.
- **Changed in the file**: the service checks `appsettings.json` every 2 seconds, takes over the new values without a restart, and the UI shows them within a few seconds (*Last changed … in appsettings.json on the server*). Event 4100 lists what changed.
- **File broken** (invalid JSON, invalid value): the current values stay in force; event 1002 and the log say what is wrong. Fix the file and save again.
- **Other sections changed** (e.g. `entra`, `configMgr`): the UI shows "Waiting for a restart: …" and event 1002 names the sections.

### Service configuration in the admin UI

The *Settings* page: **Integrations** on top, then sections that open on click, the **Notice for Administrators** at the bottom. Inside a section the values sit in groups with their own header (e.g. General: Rate Limits per Caller, Service; ConfigMgr: OS Deployments, Requests, Connection, Protection). The browser remembers which sections are open. After *Restart Service…* a new environment name shows in the sidebar without reloading the page.

| Section | Applies at once | From `appsettings.json` |
|---|---|---|
| General | rate limits | environment name, where the service listens, HTTPS, trusted proxies, public addresses, secrets encryption |
| Logging | log level | audit entries in memory, log folder, Event Log |
| ConfigMgr, Active Directory, Entra ID and Intune, Certificate Expiry | | the values of these sections |

The values from `appsettings.json` are shown with every default filled in, only to people with **Admin.Manage**: they name hosts, OUs and groups that monitoring does not need. People with Monitoring.Read see integrations, rate limits, log level and the notice. Secrets are never shown.

These keys can be changed there. They are written into `appsettings.json` and apply after a restart of the service:

| Key | In the UI |
|---|---|
| `environmentName` | General › Environment Name |
| `configMgr.osDeployments` | ConfigMgr › OS Deployments (*Add*, change, *Remove*, then *Save*) |
| `configMgr.primaryUserVariable` | ConfigMgr › Primary User Variable |
| `configMgr.requestTimeoutMs` | ConfigMgr › Timeout (in seconds, 5–300) |
| `configMgr.collectionPollTimeoutSeconds` | ConfigMgr › Collection Refresh Wait (10–1800 s) |
| `configMgr.restartOnKerberosFailure` | ConfigMgr › Restart on Kerberos Failure |
| `activeDirectory.prestageAccount` | Active Directory › Prestaged Accounts |
| `activeDirectory.timeoutMs` | Active Directory › Timeout (in seconds, 1–120) |
| `expiry.warnDays` / `expiry.intervalHours` / `expiry.iisPorts` | Certificate Expiry |
| `logging.auditBufferSize` | Logging › Audit Entries in Memory (100–100000) |

Every other key is shown read-only and can only be changed in the file on the server. That is deliberate: connections and sign-in (tenant, app IDs, hosts, LDAP URLs, certificates) and everything that decides what the API may touch or whom it trusts (`managedOus`, managed and protected groups, `reservedVariables`, allowed clients, `trustedCertificateThumbprint`, `trustProxy`, `requireHttps`, Event Log, secrets protection) need access to the server, not only a role in Entra ID. The API refuses such keys with 400 "can only be changed in appsettings.json on the server".

**Defaults:** a value that differs from its default shows *Default: …* under its name with **Reset**. Reset removes the key from `appsettings.json` (rate limits and log level: writes the default), so the service's default applies again. **Restore Defaults…** at the end of the sections resets every changed rate limit, the log level and the keys above in one step, after a confirmation that lists them; integrations, environment name, OS deployments and the notice stay as they are. Keys reset in the file apply after a restart like any other change. Through the API: `PUT /api/admin/configuration` with `null` as the value, e.g. `{"expiry.warnDays": null}`; `GET` returns the defaults in `defaults`.

`configMgr.primaryUserVariable` must not be one of `configMgr.reservedVariables` (400, also at start), and the service never writes a reserved variable, whichever route asks.

A saved key shows *After restart*, and the top of the page shows **Waiting for a restart: …** with the button **Restart Service…**. After a confirmation the service ends itself and WinSW starts it again (about 15 seconds; running calls are cut off, event **1005** names the person). The page notices when the service is back and the values are in effect. In a console (`npm start`) the button is missing; restart by hand. Keys changed by hand in the file show the same way. Event 4100 names the person and the new values of every change made in the UI.

When the UI writes the file it keeps every other key and value but sets the indentation to 2 spaces. Older versions kept log level and rate limits as overrides in `runtime.json`; the first start of this version moves them into `appsettings.json`. Templates: `appsettings.example.json`, `secrets.example.json`.

## appsettings.json

| Key | Default | Description |
|---|---|---|
| `environmentName` | `Unknown` | label in health and admin UI, e.g. `Lab`, `Production` |
| `server.host` / `server.port` | `127.0.0.1` / `5080` | where Node listens; keep it on localhost behind IIS |
| `server.trustProxy` | `["127.0.0.1", "::1"]` | peers whose `X-Forwarded-*` headers count (protocol, host, client IP): the reverse proxy. Default: IIS on the same server. `true` trusts every peer and is only safe while `server.host` is `127.0.0.1` (the service warns at start otherwise); `false` trusts none |
| `server.requireHttps` | `true` | reject plain HTTP with 403 |
| `entra.tenantId` | – | tenant ID |
| `entra.apiAppId` | – | client ID of `EndpointManagement.Service` – the token audience |
| `entra.adminUiClientId` | – | client ID of `EndpointManagement.AdminUI` – **required when the admin UI is used** ([03, part C](03-entra-setup.md#c-endpointmanagementadminui--the-ui-registration-required-for-the-ui)) |
| `entra.adminUiScope` | `api://<apiAppId>/access_as_user` | scope the UI requests; the service expects the same scope name in tokens for people |
| `entra.allowedUserClients` | `[]` | client IDs whose tokens for people are accepted. Empty = only `adminUiClientId`; with both empty no token of a person is accepted. Add another client only if people call the API through it ([03](03-entra-setup.md#how-the-service-checks-a-token)) |
| `entra.allowedAppClients` | `[]` | client IDs whose application tokens are accepted. Empty = every client with an app role assignment |
| `entra.graphClientId` | – | identity for Graph calls, usually = `apiAppId`; with `graphAuth = managedIdentity` the client ID of the user-assigned managed identity |
| `entra.graphAuth` | `certificate` | `certificate`, `secret`, or `managedIdentity` on an Azure host (App Service, Container Apps, Arc-enabled server): no certificate or secret to renew. Grant the Graph permissions of [03](03-entra-setup.md) to the managed identity |
| `entra.graphCertificatePath` | – | PEM with certificate and private key |
| `entra.managedGroups` | `[]` | Entra groups the group routes may change, patterns with `*` and `?` (e.g. `["APP_*", "CA_Pilot"]`). Empty = every group that is not protected. See [Scope](#scope-what-a-readwrite-role-may-reach) |
| `entra.protectedGroups` | `["G_EMAPI_*"]` | never changed through the API, even when they match `managedGroups`. Keep the groups that grant this API's roles in it |
| `entra.writableExtensionAttributes` | `[1, …, 15]` | device extension attributes (numbers) `PUT …/extension-attributes/{number}` may write; any other is 403, `[]` = none. Extension attributes often feed dynamic device groups and Conditional Access filters, so list only the ones clients set |
| `configMgr.enabled` | `true` | |
| `configMgr.providerHost` | `localhost` | SMS Provider FQDN |
| `configMgr.siteCode` | – | site code; read from `SMS_Site` when empty |
| `configMgr.domainShortName` | – | NetBIOS domain name for primary users (`CONTOSO\user`) and the DDR fallback |
| `configMgr.allowUntrustedCertificate` | `false` | accept any AdminService certificate – lab only; the service account's sign-in goes to whoever answers |
| `configMgr.trustedCertificateThumbprint` | – | SHA-1 thumbprint of the one AdminService certificate to accept although Windows does not trust its chain (self-signed SMS Provider certificate); safer than `allowUntrustedCertificate`, which it replaces when set |
| `configMgr.requestTimeoutMs` | `30000` | |
| `configMgr.restartOnKerberosFailure` | `true` | Windows service: restart itself when the AdminService keeps rejecting a sign-in it accepted before ([04](04-configmgr-adminservice.md)) |
| `configMgr.transport` | `windows` on Windows | `windows`: AdminService requests through the Windows HTTP stack (channel binding, needed when the AdminService enforces Extended Protection); `node`: Negotiate from Node.js |
| `configMgr.osDeployments` | `[]` | `[{ identifier, collectionName }]` – OS deployment collections per Windows version for the OSD and restaging routes |
| `configMgr.siteServer` | = `providerHost` | site server whose inbox receives DDR records |
| `configMgr.messagingSdkPath` | – | local path of `Microsoft.ConfigurationManagement.Messaging.dll` (no share); needed for the DDR after RegisterDevice ([04](04-configmgr-adminservice.md#windows-helper-process)). The helper loads it only with a valid Microsoft signature |
| `configMgr.collectionPollTimeoutSeconds` / `collectionPollIntervalMs` | `120` / `500` | wait for a running collection refresh before changing rules |
| `configMgr.reservedVariables` | see [Scope](#scope-what-a-readwrite-role-may-reach) | device variables clients may not set or remove (patterns): `API_*`, `_SMSTS*`, `SMSTSPostAction`, `OSDJoin*` and other variables that run commands or carry credentials |
| `configMgr.primaryUserVariable` | `SMSTSUdaUsers` | device variable filled by SyncPrimaryUsers |
| `configMgr.pendingPrimaryUserDirectory` | `<logging.directory>\pending-primary-users` | users SyncPrimaryUsers could not add, one file per computer |
| `activeDirectory.enabled` | `true` | |
| `activeDirectory.urls` | `[]` | `ldaps://…:636`, tried in order |
| `activeDirectory.baseDn` / `bindDn` | – | |
| `activeDirectory.managedOus` | `[]` | OUs (DNs, sub-OUs included) whose computers the API may change, and the only OUs computers may be created in or moved to. Empty = the whole `baseDn` |
| `activeDirectory.managedGroups` | `[]` | AD groups the group routes may change (patterns). Empty = every group that is not protected |
| `activeDirectory.protectedGroups` | `G_EMAPI_*`, Domain Admins, Enterprise Admins, … | never changed. Protected as well, whatever the list says: the built-in administrative groups (Domain Admins … DnsAdmins, Cert Publishers), every group with `adminCount=1` or `isCriticalSystemObject`, and every group nested in one of these |
| `activeDirectory.prestageAccount` | `randomPassword` | state of a prestaged computer account: `randomPassword` (enabled, random password nobody knows) or `disabled` ([05](05-active-directory.md#operations)) |
| `intune.enabled` | `true` | Intune routes on/off |
| `logging.level` | `info` | `fatal`, `error`, `warn`, `info`, `debug`, `trace` – live, also in the admin UI |
| `logging.directory` | – | folder for `app.log` and `audit.log`; empty = stdout only |
| `logging.auditBufferSize` | `1000` | audit entries kept in memory for the UI |
| `adminUi.staticDir` | – | folder of the built UI; empty = no UI |
| `rateLimit.enabled` | `true` | limits per caller (app registration or person), see [07](07-operations.md#rate-limits). All `rateLimit` values are live and also in the admin UI |
| `rateLimit.readsPerMinute` | `300` | GET requests per caller and minute |
| `rateLimit.changesPerMinute` | `60` | POST, PUT, DELETE per caller and minute |
| `rateLimit.criticalPerHour` | `20` | critical actions per caller and hour: delete device or computer, wipe, retire, BitLocker recovery key |
| `eventLog.enabled` | `true` | write to the Windows Event Log *Endpoint Management API* ([07](07-operations.md#windows-event-log)); ignored outside Windows |
| `secretsProtection.protect` | `true` | encrypt plain values in `secrets.json` with Windows DPAPI at start (see below) |
| `secretsProtection.scope` | `user` | `user`: only the service account can decrypt; `machine`: every account on this server can |
| `expiry.warnDays` | `30` | warn this many days before a certificate or secret expires |
| `expiry.intervalHours` | `6` | how often expiry is checked |
| `server.publicUrls` | `[]` | addresses clients use, e.g. `["https://endpoint-management.contoso.local"]`. When set, exactly these are watched. When empty, the service learns them from successful HTTPS requests of people or clients with `Admin.Manage` or `Monitoring.Read` on the `expiry.iisPorts` (DNS names only, kept in `public-hosts.json`, forgotten after 30 days without a request). Their HTTPS certificate is watched for expiry |
| `expiry.iisPorts` | `[443]` | IIS HTTPS bindings on these ports are checked. Other ports on the same server usually belong to other services (WSUS 8531, Web Management Service 8172, ConfigMgr site systems); add them if you want to watch those too, `[]` = none |
| `onPrem.mode` | `local` | `local`: ConfigMgr and Active Directory run in this process. `loopback`: also in this process, but every call goes through the remote layer ([01](01-architecture.md#configmgr-and-active-directory-behind-ports)); for testing that layer against the real systems |
| `onPrem.callTimeoutSeconds` | `300` | longest a ConfigMgr or AD call may take behind the remote layer, collection polling included; then 504 |

## Scope: what a ReadWrite role may reach

The app roles say *what* a caller may do; the scope settings say *on which objects*. The service account can do much more than any single client should (`GroupMember.ReadWrite.All` covers every group in the tenant, the LDAP account has rights on whole OUs), so without these lists every holder of `Entra.ReadWrite` or `AD.ReadWrite` could reach all of it, for example add themselves to `G_EMAPI_Admins` and get every role.

| Always, whatever the settings | |
|---|---|
| Entra and AD groups in `protectedGroups` | 403. Default `G_EMAPI_*` (the groups of [03, B2](03-entra-setup.md#b2-create-the-groups)); AD also Domain Admins, Enterprise Admins, Schema Admins, Administrators, Account/Backup/Server/Print Operators, DnsAdmins, Key Admins, Cert Publishers, Protected Users and the domain controller groups |
| AD groups with `adminCount=1` or built-in groups | 403 (groups protected by AdminSDHolder) |
| AD groups nested in a protected group | 403: a member would get the rights of the outer group (e.g. a group inside DnsAdmins). Checked with the in-chain rule of AD over every level |
| Domain controllers, read-only DCs, (group) managed service accounts | 403 for every change, and LAPS and BitLocker reads; reading their info still works |
| Task sequence variables in `configMgr.reservedVariables` | 403 for set and remove; a replace of all variables keeps them on the device |
| Several objects with the same name | 409 instead of picking one ([09](09-api-reference.md)); an Entra device that is the only hybrid joined one, and an Intune device that is the only company-owned one, still win (see below) |

| Recommended for production | Example |
|---|---|
| `entra.managedGroups` | `["APP_*", "DEV_*", "CA_Pilot"]` – only the groups clients really manage |
| `activeDirectory.managedOus` | `["OU=Clients,DC=contoso,DC=local"]` – the same OUs as the delegation in [05](05-active-directory.md#service-account) |
| `activeDirectory.managedGroups` | `["APP_*", "GPO_*"]` |

Default reserved variables: `API_*`, `_SMSTS*`, `SMSTSPostAction`, `SMSTSPreferredAdvertID`, `SMSTSRunCommandLine*`, `SMSTSRunPowerShell*`, `SMSTSPeerRequestPort`, `OSDJoin*`, `OSDDomainName`, `OSDDomainOUName`, `OSDLocalAdminPassword`, `OSDCaptureAccount*`, `OSDRandomAdminPassword`, `OSDAdminPassword`, `OSDNetworkJoin*`, `SMSConnectNetworkFolder*`, `SMSInstallUpdateTarget`. Ordinary OSD settings (`OSDComputerName`, `OSDLanguage`, `OSDTimeZone` …) stay allowed. Setting the list replaces the default, so copy it and add to it.

**Your own group names:** `G_EMAPI_*` is only the default, after the names proposed in [03, B2](03-entra-setup.md#b2-create-the-groups). If the groups that grant this API's roles are called differently, put their names into `entra.protectedGroups` (cloud groups) and `activeDirectory.protectedGroups` (groups from AD), e.g. `["APP-EMAPI-*", "SEC_EndpointAPI_Admins"]`. A list you set replaces the default; in AD the built-in administrative groups (including DnsAdmins, Group Policy Creator Owners and Cert Publishers, which carry no `adminCount`) stay protected anyway, whatever the list says.

All scope settings are read at start; change `appsettings.json` and restart the service.

Not covered by these settings, so keep them in mind when you grant roles:

- **ConfigMgr collections and devices:** `ConfigMgr.ReadWrite` reaches every collection and device the service account's ConfigMgr security role covers; `configMgr.osDeployments` only names the OS deployment collections for the OSD routes and is no limit for *Add to Collection*. Give the account a custom security role with a security scope of the managed collections ([04](04-configmgr-adminservice.md)).
- **Entra ID and Intune devices:** enable/disable, retire, wipe and delete act on any device of that name (servers and privileged access workstations included) once the name is unique. Keep those roles for the people and clients that need them.
- **Device attributes that drive groups:** extension attributes (limited by `entra.writableExtensionAttributes`), the Autopilot group tag and the Intune device category can be inputs of dynamic device groups, Conditional Access filters and Intune assignments. A caller with `Entra.ReadWrite` or `Intune.ReadWrite` that sets them may move a device into such a group although the group routes would refuse it. Check your dynamic rules before you grant these roles.
- **Same name twice:** when two Intune devices share a name and exactly one is company-owned, that one is used; when two Entra devices share a name and exactly one is hybrid joined, that one is used. This is on purpose (the usual leftover after a reinstall or hybrid join), but the caller cannot choose the other record. If people enroll personal devices, keep Wipe and Retire for company-owned devices, or remove stale records so names stay unique.
- **Prestaged computer accounts** that never join stay in AD with their random password; review them now and then (`pwdLastSet` unchanged, no `lastLogonTimestamp`).
- **`secretsProtection.scope` `machine`:** every account on the server can decrypt `secrets.json` once it can read it; the folder ACL is then the only protection. Prefer `user`.
- **A request that runs long:** the collection refresh wait stops when the caller disconnects; a request already sent to ConfigMgr or AD (e.g. a full policy reset) still runs to its own timeout (`requestTimeoutMs`).

## secrets.json

All values can also be set as environment variables on the service (they win over the file).

| Key | Environment variable | Needed when |
|---|---|---|
| `graphClientSecret` | `EMAPI_GRAPH_CLIENT_SECRET` | `entra.graphAuth = secret` |
| `ldapBindPassword` | `EMAPI_LDAP_BIND_PASSWORD` | Active Directory enabled |

The installer limits the `config` folder to administrators, SYSTEM and the service account.

### Encryption with DPAPI

Write the values in plain text. At the next start the service encrypts every plain value with Windows DPAPI and rewrites the file, so it looks like this afterwards:

```json
{ "ldapBindPassword": "dpapi:AQAAANCMnd8BFdERjHoAwE/Cl+sBAAAA…" }
```

- Event 4200 in the event log names the keys that were encrypted.
- With `secretsProtection.scope = user` (default) the value is bound to the **service account on this server**. Another account or another server cannot read it – not even an administrator. So **write the file in plain text and let the service encrypt it**; do not copy an encrypted `secrets.json` to another server.
- **Changing one value** (new LDAP password, new Graph secret): replace only that `dpapi:…` string with the new plain value and restart the service. The other `dpapi:` values can stay; plain and encrypted values may be mixed.
- **Changing the service account or moving to another server: replace every `dpapi:…` value with plain text.** The old ones were encrypted for the old account or server and cannot be read; one unreadable value is enough for the service to stop at once (`err.log`: "Key not valid for use in specified state", IIS: 502). Steps in [02, Changing the service account](02-installation.md#changing-the-service-account).
- `scope = machine` is for a setup where several accounts on the same server must read it; it protects against copying the file to another machine, not against other accounts on this one.
- Environment variables (`EMAPI_…`) are not encrypted by the service; prefer the file.

## runtime.json (admin UI)

| Setting | Effect |
|---|---|
| `areas.ConfigMgr / Ad / Intune / Entra` | `false` = all routes of that area answer 503 "disabled by an administrator" |
| `notice` | banner on the status page for all administrators |

## sequences.json (admin UI)

Saved action sequences of the device page (*Run Sequence*), changed in the admin UI by people with `Admin.Manage`. Same folder, written atomically one save after another, no restart needed. A damaged file (e.g. edited by hand) does not stop the service: it starts without sequences, keeps the file as `sequences.json.damaged-<time>` and writes event 1002. See [07](07-operations.md#sequences).

Changes are written atomically and logged with the name of the person (`runtime settings changed`).
