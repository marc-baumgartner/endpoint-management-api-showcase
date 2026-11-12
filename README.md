<h1><img src="docs/images/app-icon.svg" alt="" width="36" align="absmiddle">&nbsp;Endpoint Management API</h1>

> **Portfolio showcase – documentation, screenshots and a live demo.** The source code is private. This repository shows what the product does and how it is built and operated. No license is granted (see [LICENSE](LICENSE)); all IDs, host names and domains are placeholders (`contoso.local`, `00000000-…`).

One REST API for **ConfigMgr, Active Directory, Intune and Entra ID**, protected by Entra ID app roles. Task sequences, ITSM tools (ServiceNow & co.) and scripts call it with their own app registration – through the **PowerShell module** or directly over **REST**. People use the **admin UI** in the browser with their work account.

> [!TIP]
> ### [Try the admin UI live in your browser](https://marc-baumgartner.github.io/endpoint-management-api/)
> The real admin UI with sample data, no sign-in needed. Under *Devices*, pick one of the listed sample devices, for example **CH-NB-04711**. Actions change nothing, and nothing is sent anywhere.

<p>
  <a href="docs/videos/admin-ui-tour.mp4"><img src="docs/images/admin-ui-tour.gif" alt="Admin UI tour: status, device record, actions, remediation picker, LAPS password, greyed-out actions for a device that is not everywhere, recent devices and a link to the device's audit calls, copying with a toast, a sequence built, saved and run, audit log, routes, settings with defaults, dark mode"></a>
  <br>
  <sub><b>Video:</b> a two-minute tour of the admin UI in demo mode at double speed. It shows the status page, a device record with its actions, a remediation picked from the tenant's list, a local admin password, actions greyed out with the reason for a device that is only in Autopilot or has no AD account, a sequence that is built, saved and run, the audit log, the route catalogue, the settings with a value reset to its default and a restart of the service, and dark mode. <a href="docs/videos/admin-ui-tour.mp4">Full resolution (MP4)</a></sub>
</p>

## At a glance

| Part | Technology |
|---|---|
| Service | Node.js 24 LTS (TypeScript, Fastify) as Windows service, IIS as HTTPS reverse proxy |
| ConfigMgr | AdminService (REST on the SMS Provider), Windows sign-in as the service account |
| Active Directory | LDAPS |
| Intune / Entra ID | Microsoft Graph |
| Callers | Entra ID app roles: application permissions for clients, group assignments for people |
| Admin UI | React + Base UI: service status, device record with actions and action sequences, audit log, settings |
| Client | PowerShell module (version 2.0; Windows PowerShell 5.1, PowerShell 7, WinPE) |
| Docs | the [docs](docs) folder of this repository + OpenAPI at `/api/docs` in the product |

**91 routes** in five areas – ConfigMgr (28), Active Directory (20), Intune (23), Entra ID (18), Monitoring (2) – all with the same response envelope and role rule ([09](docs/09-api-reference.md)).

## Architecture

**Admin UI.** People sign in with Entra ID; the service calls Microsoft Graph with its own app registration and reaches ConfigMgr and Active Directory directly.

![Admin UI: browser, IIS and the service; sign-in through Entra ID; Graph permissions of the service; example requests to Entra ID, Autopilot, Intune, ConfigMgr and Active Directory](docs/images/architecture-admin-ui.png)

**Clients.** Task sequences, ServiceNow, scripts and monitoring each have their own app registration, get a token from Entra ID and call the API over HTTPS. Every call is checked for token and role and written to the audit log.

![Clients: task sequence, ServiceNow, scripts and monitoring, each with its own app registration, get a token from Entra ID and call the API; app roles and example requests](docs/images/architecture-clients.png)



## Permission model

App roles live on one API registration. Each client system has its own app registration with application permissions; people get roles through Entra groups. Example assignment:

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/images/permissions-role-matrix-dark.png">
  <img src="docs/images/permissions-role-matrix-light.png" alt="Permission matrix: client app registrations and Entra groups against the app roles">
</picture>


## Admin UI

Status of every connection, a device record across all four systems with the actions the signed-in person's roles allow, audit log, route catalogue and runtime settings. React + Base UI, light and dark appearance, works on phone width. (Screenshots from the built-in demo mode with sample data.)

### Status and device record

The status page shows every connection and what expires soon. The device record shows one device in all four systems on one page.

| | |
|---|---|
| ![Status: connections, expiring certificates and secrets, traffic of the last hour, areas](docs/images/admin-ui-status.png)<br><sub>Status page: connection to each system, expiring certificates and secrets, traffic of the last hour, areas on or off</sub> | ![Device record across ConfigMgr, AD, Intune and Entra ID with the actions menu](docs/images/admin-ui-device-record-actions-menu.png)<br><sub>Device record: one device in ConfigMgr, AD, Intune and Entra ID, actions menu open</sub> |

### Actions

Actions are grouped by system and by what they do. Every action asks only for what it needs; destructive actions ask for the device name before they run. A value an action changes shows on the page at once, even while Entra ID still reports the old one.

| | |
|---|---|
| ![Sheet: add to collection](docs/images/admin-ui-action-add-to-collection.png)<br><sub>Add the device to a ConfigMgr collection</sub> | ![Intune actions: rename, Autopilot group tag, remediation on demand, Defender scan, LAPS rotation](docs/images/admin-ui-intune-actions-menu.png)<br><sub>Intune menu: rename, Autopilot group tag, remediation, Defender scan, LAPS rotation</sub> |
| ![Remediation picked from the list of the tenant](docs/images/admin-ui-remediation-picker.png)<br><sub>Run a remediation: picked from the tenant's list</sub> | ![Confirmation before an Intune wipe](docs/images/admin-ui-action-wipe-confirmation.png)<br><sub>Intune wipe: runs only after the device name is typed</sub> |
| ![Sheet: set extension attribute 5](docs/images/admin-ui-extension-attribute.png)<br><sub>Set extension attribute 5 in Entra ID</sub> | ![Extension attribute 5 on the device page right after saving](docs/images/admin-ui-extension-attribute-saved.png)<br><sub>The new value shows on the device page right after saving</sub> |

### Passwords and recovery keys

LAPS passwords and BitLocker recovery keys need roles of their own, are shown only on request and are logged in the audit log.

| | |
|---|---|
| ![Local admin password from Windows LAPS, shown on demand and logged as a critical action](docs/images/admin-ui-laps-password.png)<br><sub>Local admin password (Windows LAPS): shown on request, logged in the audit log</sub> | ![BitLocker recovery key from Entra ID, matched by its key ID](docs/images/admin-ui-bitlocker-key.png)<br><sub>BitLocker recovery key from Entra ID, found by its key ID</sub> |

### Greyed-out actions

Nothing disappears from a menu. Actions a person lacks the role for are greyed out with the role they need; actions that cannot run on this device are greyed out with the reason (no computer account, not enrolled, already exists, nothing stored).

| | |
|---|---|
| ![Active Directory menu: account, groups, security; BitLocker greyed out without AD.BitLocker.Read](docs/images/admin-ui-ad-actions-missing-role.png)<br><sub>AD menu: BitLocker greyed out, the missing role is named</sub> | ![A serial number only registered in Windows Autopilot: only the Autopilot actions can run, the rest is greyed out with Not enrolled](docs/images/admin-ui-autopilot-only.png)<br><sub>Device only in Autopilot: Autopilot actions run, the rest is greyed out</sub> |
| ![No computer account in AD: only Create Computer Account can run, the rest is greyed out](docs/images/admin-ui-actions-greyed-not-found.png)<br><sub>No AD account: only Create Computer Account runs, the reason is shown</sub> | ![No computer account in AD: create it in an OU](docs/images/admin-ui-create-computer-account.png)<br><sub>No AD account yet: create it in a chosen OU</sub> |

### Sequences

Device actions as building blocks: sync in Intune, wait, set the primary user in Intune and ConfigMgr, refresh policy – one after another, each step with the signed-in person's roles and in the audit log. Saved sequences are shared; fields left empty (the user name) are asked for each time. Steps follow the same greying rules as the menus.

| | |
|---|---|
| ![Building a sequence from saved "Hand Over to User"](docs/images/admin-ui-sequence-build.png)<br><sub>Building the saved sequence "Hand Over to User", step by step</sub> | ![Every step done, with its result](docs/images/admin-ui-sequence-done.png)<br><sub>Sequence finished: each step with its result</sub> |
| ![Add Step in a sequence: AD actions greyed out until a step creates the computer account](docs/images/admin-ui-sequence-steps-greyed.png)<br><sub>Adding a step: AD steps wait for a step that creates the account</sub> | ![A saved sequence changed: Save, Delete and Changes not saved](docs/images/admin-ui-sequence-edit.png)<br><sub>Saved sequence changed: Save or Delete</sub> |
| ![Closing a sequence with steps that are not saved asks: Keep Editing or Discard](docs/images/admin-ui-sequence-unsaved.png)<br><sub>Closing an unsaved sequence: Keep Editing or Discard</sub> |  |

### Audit log and routes

| | |
|---|---|
| ![Audit log with caller, route, status and errors](docs/images/admin-ui-audit-log.png)<br><sub>Audit log: who called which route, status, errors</sub> | ![Route catalogue with the roles per route](docs/images/admin-ui-route-catalogue.png)<br><sub>Route catalogue: all 91 routes with the role each needs</sub> |

### Settings

Integrations on top, everything else in sections. Settings from `appsettings.json` apply after a restart from the same page.

| | |
|---|---|
| ![Runtime settings](docs/images/admin-ui-settings.png)<br><sub>Settings: integrations on and off, other settings in sections, Restore Defaults</sub> | ![Settings from appsettings.json in sections, waiting for a restart](docs/images/admin-ui-settings-configuration.png)<br><sub>appsettings.json in the UI: a changed value shows its default with Reset, applies after a restart</sub> |
| ![Turning an integration off asks first](docs/images/admin-ui-settings-turn-off-integration.png)<br><sub>Turning off an integration asks for confirmation first</sub> |  |

### Notices

Short notices appear at the bottom and never move the page.

| | |
|---|---|
| ![Serial number copied: a toast at the bottom, nothing on the page moves](docs/images/admin-ui-toast-copy.png)<br><sub>Serial number copied: short notice at the bottom</sub> | ![The API cannot be reached and the sign-in has expired: toasts with Sign In Again](docs/images/admin-ui-toast-connection.png)<br><sub>API unreachable and sign-in expired: notices with Sign In Again</sub> |

### Account and dark mode

Light, dark or automatic, switched in the account menu. Under *Roles and Session* the menu shows when the person signed in, how long the token is valid, the tenant and every app role, with the assigned ones ticked.

| | |
|---|---|
| ![Account menu with roles and session opened](docs/images/admin-ui-account-menu.png)<br><sub>Account menu: appearance, signed-in roles, session details</sub> | ![Account menu, dark](docs/images/admin-ui-account-menu-dark.png)<br><sub>Account menu in dark mode</sub> |
| ![Status, dark](docs/images/admin-ui-status-dark.png)<br><sub>Status page in dark mode</sub> | ![Device record, dark](docs/images/admin-ui-device-record-dark.png)<br><sub>Device record in dark mode</sub> |
| ![Audit, dark](docs/images/admin-ui-audit-log-dark.png)<br><sub>Audit log in dark mode</sub> |  |

### Phone

| | | |
|---|---|---|
| ![Status on a phone](docs/images/admin-ui-phone-status.png)<br><sub>Status page on a phone</sub> | ![Device record on a phone](docs/images/admin-ui-phone-device-record.png)<br><sub>Device record on a phone</sub> | ![Sequence run on a phone](docs/images/admin-ui-phone-sequence.png)<br><sub>Sequence run on a phone</sub> |

## API and clients

The screenshots below show the real service code and PowerShell module, running locally against simulated ConfigMgr, AD, Intune and Entra ID back ends with sample data.

### OpenAPI description and requests from a REST client

The service publishes its OpenAPI description at `/api/docs`; the same page works as a REST client (Swagger UI, *Try it out*) with a bearer token.

| | |
|---|---|
| ![OpenAPI description of all routes](docs/images/api-openapi-description.png) | ![GET request: computer info from Active Directory](docs/images/api-rest-get-computer-info.png) |
| ![POST request: add a device to a ConfigMgr collection](docs/images/api-rest-post-add-to-collection.png) | |

### PowerShell module

One cmdlet per route. Errors come back with the HTTP status and a readable message.

![PowerShell session: device lookups, collection change, health, a 404](docs/images/powershell-module-session.png)

A client with read-only roles can look things up, but changes are refused with 403 and the missing role:

![PowerShell session with a read-only client: 403 for a change](docs/images/powershell-read-only-client-403.png)

Sequences saved in the admin UI run through the API as well, step by step with the client's own roles. The same session sets an Autopilot group tag, lists the remediations of the tenant and reads a LAPS password:

![PowerShell: run the saved sequence "Hand Over to User", Autopilot group tag, remediations, LAPS](docs/images/powershell-sequences-and-new-cmdlets.png)

A client without `ConfigMgr.ReadWrite` is refused before the first step runs, with the steps whose role is missing:

![PowerShell: the sequence is refused with 403, steps 4 and 5 need ConfigMgr.ReadWrite](docs/images/powershell-sequence-refused-403.png)

### Automated tests

Contract, authentication, validation and connector tests run on every push (the Windows-only tests of the helper process run in the Windows CI build).

![Test run](docs/images/tests-run.png)

## Permissions at a glance

| App registration | Who | Gets |
|---|---|---|
| `EndpointManagement.Service` | the API itself | defines the app roles; its identity calls Graph |
| `EndpointManagement.AdminUI` | the admin UI in the browser | delegated scope `access_as_user` – people's rights come from their Entra groups |
| one per client, e.g. `EndpointManagement.Client.TaskSequence`, `EndpointManagement.Client.ServiceNow` | task sequences, ITSM, scripts | only the app roles that client needs (application permissions) |

Step by step: [03 Entra ID setup](docs/03-entra-setup.md). Connecting a client: [10 Clients](docs/10-clients.md).

## How it is built

| Area | Approach |
|---|---|
| Code | TypeScript throughout: Fastify service, connectors per system (AdminService, LDAP, Graph), one service class per system, route contract with all 91 routes and their roles |
| Quality | contract tests (every route registered with its roles), token and role tests, connector tests against simulated Graph/AdminService/LDAP, PowerShell syntax check under Windows PowerShell 5.1 in CI |
| Security | Entra ID access tokens only (no API keys), least-privilege app roles per client, dedicated roles for wipe and BitLocker recovery keys, rate limits per caller (429 with Retry-After), HTTPS only, audit log of every call, secrets encrypted with Windows DPAPI |
| Operations | Windows service (WinSW) under a gMSA, IIS as reverse proxy, own Windows Event Log with fixed event IDs, expiry warnings for certificates and client secrets, health endpoint, runtime switches per area without restart, one-command install and update with rollback |
| Admin UI | React 19, Base UI components, MSAL (PKCE), Apple Human Interface Guidelines as design reference, light and dark appearance |

## Documentation

| | |
|---|---|
| [01 Architecture](docs/01-architecture.md) | components, request flow, design decisions |
| [02 Installation](docs/02-installation.md) | server prerequisites, install, update, rollback |
| [03 Entra ID setup](docs/03-entra-setup.md) | **permission overview**, app registrations, app roles, admin UI sign-in, role assignment via Entra groups |
| [04 ConfigMgr and AdminService](docs/04-configmgr-adminservice.md) | Windows sign-in, gMSA, ConfigMgr rights, Windows helper, AdminService calls |
| [05 Active Directory](docs/05-active-directory.md) | LDAPS, service account, delegated permissions, LDAP operations |
| [06 Configuration](docs/06-configuration.md) | every setting, secrets, runtime settings |
| [07 Operations](docs/07-operations.md) | admin UI, sequences, Windows Event Log IDs, expiry warnings, rate limits, logs, health, troubleshooting |
| [08 PowerShell module](docs/08-powershell-module.md) | install, sign-in, task sequences, cmdlets |
| [09 API reference](docs/09-api-reference.md) | all routes with roles and cmdlets |
| [10 Clients](docs/10-clients.md) | connecting task sequences, ServiceNow and scripts – PowerShell module or REST |
