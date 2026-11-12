# 02 – Installation

## Before you start

| Requirement | Notes |
|---|---|
| Windows Server 2019 or later | the ConfigMgr site server or a member server with network access to the SMS Provider |
| IIS with **URL Rewrite 2.1** and **Application Request Routing 3.0** | IIS only forwards; no ASP.NET needed |
| TLS certificate in `LocalMachine\My` | **required** – the API and the admin UI run on HTTPS only; see [TLS certificate](#tls-certificate-https) |
| Service account | a gMSA such as `CONTOSO\gmsa-emapi$` (recommended, installed on the server with `Install-ADServiceAccount`) or a domain account such as `CONTOSO\svc-emapi`; see [04](04-configmgr-adminservice.md). Not a personal admin account: the service and Kerberos to ConfigMgr run under it. |
| LDAP service account, e.g. `svc-emapi-ldap` | see [05](05-active-directory.md); all accounts at a glance under [Accounts](#accounts) |
| Entra app registrations | `EndpointManagement.Service`, `EndpointManagement.AdminUI` and one per client, see [03](03-entra-setup.md) |
| Release | the **EndpointManagementAPI** artifact of the GitHub Actions build; contains Node.js and WinSW, so nothing else needs installing. The 5 newest builds stay downloadable (each at most 7 days); for an older one, run the build again (*Actions → build → Run workflow*). A version released on GitHub (*Releases → Draft a new release*, new tag `v<version>` equal to `apps/api/package.json`) gets the same zip as **EndpointManagementAPI.zip** under *Assets*, with the manifest hash in its notes; releases do not count against the Actions storage and stay. The *Code → Download ZIP* button gives the source code, which cannot be installed. |

## Accounts

Three identities, each with its own job; none of them is a personal admin account.

| Identity | Type | Used for | Set up in |
|---|---|---|---|
| `CONTOSO\gmsa-emapi$` | gMSA (or a domain account `CONTOSO\svc-emapi`) | runs the Windows service: ConfigMgr AdminService (Kerberos), status messages over WMI, Register Device (DDR into `ddm.box`), full policy reset on clients, decrypting encrypted Windows LAPS, DPAPI of `secrets.json`, logs | [04](04-configmgr-adminservice.md#authentication-windows-sign-in-as-the-service-account) |
| `svc-emapi-ldap` | normal user with password | LDAP to Active Directory: read, write, delegated rights on the managed OUs | [05](05-active-directory.md#service-account) |
| `EndpointManagement.Service` | Entra app registration (secret or certificate) | Microsoft Graph: Intune and Entra ID | [03](03-entra-setup.md) |

The IIS site runs as its own app pool identity (`IIS AppPool\EndpointManagementAPI-Proxy`) with anonymous access; it only forwards to the service and needs no rights in AD, ConfigMgr or Entra ID. Who may do what is decided by the person's or client's Entra token (app roles), not by these accounts; the audit log records who started each action.

Descriptions for the objects in AD (optional, they help the next administrator):

| Object | Description |
|---|---|
| `gmsa-emapi` | Endpoint Management API: service account of the Windows service (ConfigMgr AdminService, WMI, LAPS decryption) |
| `G_EMAPI_ConfigMgr` | Endpoint Management API: administrative user in ConfigMgr; members are the service accounts of the API |
| `svc-emapi-ldap` | Endpoint Management API: LDAP bind account, delegated rights on the managed OUs, no interactive sign-in |

### Changing the service account

For example from a domain account to the gMSA. No uninstall is needed:

1. Run the installer again with the new account and `-SiteMode none` (IIS stays as it is): `.\deploy\windows\Install-EndpointManagementApi.ps1 -ServiceAccount 'CONTOSO\gmsa-emapi$' -SiteMode none`. It keeps `appsettings.json`, `sequences.json` and the logs and gives the new account its rights.
2. In `config\secrets.json` replace **every** `dpapi:…` value with its plain text. They were encrypted for the old account; the new one cannot read them and the service stops at once ("Key not valid for use in specified state", IIS answers 502). See [06](06-configuration.md#encryption-with-dpapi).
3. Give the new account its rights in ConfigMgr ([04](04-configmgr-adminservice.md)), on `ddm.box` and for LAPS, then `Start-Service EndpointManagementAPI`.
4. Check: `Get-CimInstance Win32_Service -Filter "Name='EndpointManagementAPI'" | Select-Object StartName, State` and *Status* in the admin UI.

If the service does not start, the reason is in `logs\service\EndpointManagementAPI.err.log` (errors before the service log is ready) or `logs\app.log`. The service restarts itself after a crash, so a fixed `secrets.json` is often picked up without a manual start.

## TLS certificate (HTTPS)

The API and the admin UI only work over **HTTPS**:

- the service answers plain HTTP with 403 (`server.requireHttps`) – clients send access tokens, these must not travel unencrypted,
- Entra ID accepts only `https://` redirect URIs for the admin UI (except `http://localhost` for development),
- the browser runs the sign-in code (PKCE, Web Crypto) only on secure pages.

TLS ends in **IIS**; the Node.js service itself listens on `http://127.0.0.1:5080` and is not reachable from outside. So the certificate is bound to the IIS site, not to the service.

### Requirements

| Requirement | Why / how to check |
|---|---|
| **Every host name people and clients use is in the Subject Alternative Name (SAN)**, e.g. `sccm01.contoso.local` **and** `endpoint-management.contoso.local` | browsers and PowerShell check the name in the address against the SAN; a name that is missing gives a certificate error. *Details → Subject Alternative Name* |
| Enhanced Key Usage **Server Authentication** (`1.3.6.1.5.5.7.3.1`) | web server template of the internal CA |
| **Private key** present, in *Local Computer → Personal* (`Cert:\LocalMachine\My`) | key symbol on the certificate icon; the installer checks it |
| Issued by a CA the **callers trust** | domain PCs and browsers trust the internal CA through GPO. **WinPE** only trusts what is in the boot image: add the root CA to the boot image or task sequences fail with "could not establish trust relationship" |
| Valid, RSA 2048 bit or more, SHA-256 signature | *General* tab; *Thumbprint algorithm sha1* in the dialog is only how Windows computes the thumbprint, not the signature |

Check on the server:

```powershell
Get-ChildItem Cert:\LocalMachine\My | Where-Object HasPrivateKey |
    Select-Object Thumbprint, NotAfter, @{n='Names';e={$_.DnsNameList.Unicode -join ', '}}, @{n='EKU';e={$_.EnhancedKeyUsageList.FriendlyName -join ', '}}
```

### Requesting the certificate from the internal CA

The certificate needs **both** names in the SAN: the server name **and** the alias (e.g. `sccm01.contoso.local` and `endpoint-management.contoso.local`). The ConfigMgr templates (e.g. for IIS/MP/DP) build the subject from Active Directory and do **not** let you add further names. Create a dedicated template once:

**1. Template** – on the CA: `certtmpl.msc`

1. Right-click **Web Server** → *Duplicate Template*.
2. *General*: name `EndpointManagement WebServer`, validity e.g. 2 years.
3. *Request Handling*: tick *Allow private key to be exported* only if you want to move the certificate later.
4. *Cryptography*: minimum key size **2048**, provider/algorithm RSA, request hash **SHA256**.
5. *Subject Name*: **Supply in the request** (this is what allows the SAN entries).
6. *Extensions → Application Policies*: **Server Authentication** (already there from Web Server).
7. *Security*: the server's computer account (e.g. `SCCM01$`) or the admins who request it: **Read** and **Enroll**.
8. *OK*.

**2. Publish** – `certsrv.msc` → *Certificate Templates* → right-click → *New → Certificate Template to Issue* → `EndpointManagement WebServer`.

**3. Request on the API server** – `certlm.msc` (Local Computer) → *Personal → Certificates* → right-click → *All Tasks → Request New Certificate* → Active Directory Enrollment Policy → tick `EndpointManagement WebServer` → *More information is required … click here*:

| Tab | Type | Value |
|---|---|---|
| Subject | Common name | `endpoint-management.contoso.local` |
| Subject | Alternative name **DNS** | `endpoint-management.contoso.local` → *Add* |
| Subject | Alternative name **DNS** | `sccm01.contoso.local` → *Add* |
| General | Friendly name | `EndpointManagement API` |

*OK* → *Enroll*. The certificate lands in `Cert:\LocalMachine\My` with its private key.

Or from the command line (same template):

```powershell
@"
[NewRequest]
Subject = "CN=endpoint-management.contoso.local"
KeyLength = 2048
KeySpec = 1
MachineKeySet = TRUE
Exportable = FALSE
FriendlyName = "EndpointManagement API"
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=endpoint-management.contoso.local&"
_continue_ = "dns=sccm01.contoso.local"
[RequestAttributes]
CertificateTemplate = EndpointManagementWebServer
"@ | Set-Content C:\Temp\emapi.inf -Encoding ASCII
certreq -new C:\Temp\emapi.inf C:\Temp\emapi.req
certreq -submit C:\Temp\emapi.req C:\Temp\emapi.cer     # choose the CA
certreq -accept C:\Temp\emapi.cer
```

`CertificateTemplate` is the template **name without spaces** (*General* tab, "Template name"), not the display name.

**4. DNS** – the alias needs a DNS record pointing to the server, e.g. CNAME `endpoint-management` → `sccm01.contoso.local` (or an A record).

**5. Bind** – install or re-run `Configure-IISProxy.ps1` with the new thumbprint (`-CertificateThumbprint`), or pick the certificate in *IIS Manager → site → Bindings → https*.

### Host name, certificate and redirect URI belong together

The address in the browser must be the same in three places:

| Place | Example |
|---|---|
| certificate SAN | `endpoint-management.contoso.local` |
| IIS binding (host name of the site, HTTPS 443) | `endpoint-management.contoso.local` |
| redirect URI of `EndpointManagement.AdminUI` ([03, part C](03-entra-setup.md#c-endpointmanagementadminui--the-ui-registration-required-for-the-ui)) | `https://endpoint-management.contoso.local/` |

If people also open the UI with the server name (`https://sccm01.contoso.local/EndpointManagementAPI/`), that name must be in the SAN **and** registered as a second redirect URI. Clients (PowerShell module, ServiceNow) use the same `BaseUrl` rules: the name in `BaseUrl` must be in the SAN.

### Binding

- `-SiteMode new` (own site): the installer creates the HTTPS binding with `-HostName` and `-CertificateThumbprint`.
- `-SiteMode existing` (application below e.g. *Default Web Site*): the site's existing HTTPS binding and its certificate are used – make sure that certificate has the names above.

### Renewing the certificate

Enroll the new certificate (same names), then in *IIS Manager → site → Bindings → https → Edit* pick the new certificate – or run `Configure-IISProxy.ps1` again with the new `-CertificateThumbprint`. The service needs no change and no restart.

Not to be confused with the certificates the service **trusts** for its own outbound connections (AdminService, LDAPS): those only need the issuing CA in the server's *Trusted Root Certification Authorities* ([05](05-active-directory.md#connection)).

## Install

1. Download the release: GitHub → *Actions* → the latest successful **build** run on `main` → *Artifacts* → **EndpointManagementAPI**. The run's summary shows the line *Release manifest SHA-256: …*; note it. (Pull request builds are called *EndpointManagementAPI-pull-request*; do not install those.)
2. Unblock and unpack it in an administrator PowerShell, into a folder only administrators can change:

   ```powershell
   Unblock-File .\EndpointManagementAPI.zip
   New-Item C:\EMAPI-Release -ItemType Directory   # from an elevated PowerShell, directly below C:\
   Expand-Archive .\EndpointManagementAPI.zip C:\EMAPI-Release
   cd C:\EMAPI-Release
   ```

   Without `Unblock-File`, Windows marks every script as downloaded from the internet and an execution policy of `RemoteSigned` refuses to run them.

   **What the installer checks:** every file must match `release-manifest.json` (SHA-256, no extra and no missing files). The release is first copied to `C:\Program Files\EndpointManagementAPI.new`, where only administrators can change it, and checked there; only a release that passes replaces the installed one, so a failed check leaves the running version untouched. It refuses a release folder whose files or folders any account other than administrators, SYSTEM, TrustedInstaller and you may change, or that an account outside the local Administrators group owns, the parent folders included (e.g. unpacked into a share, `C:\Users\Public`, or a `C:\Temp` someone created without elevation); that users may create new files in a folder, as in `C:\Temp`, is fine. It prints the manifest's SHA-256: it must be the one in the summary of the build run. Pass it as `-ManifestSha256 '<hash>'` and the installer compares it for you and stops on a difference. `-AllowWritableReleaseFolder` installs anyway after a warning.

3. Run as administrator:

   ```powershell
   # Application below an existing site: https://<server>/EndpointManagementAPI/
   .\deploy\windows\Install-EndpointManagementApi.ps1 -ServiceAccount 'CONTOSO\gmsa-emapi$' `
       -SiteMode existing -SiteName 'Default Web Site' -VirtualPath '/EndpointManagementAPI'

   # or: a site of its own, run as a domain account (the script asks for its password)
   .\deploy\windows\Install-EndpointManagementApi.ps1 -ServiceAccount 'CONTOSO\svc-emapi' `
       -SiteMode new -HostName 'endpoint-management.contoso.local' -CertificateThumbprint 'a1 b2 c3 ...'
   ```

   - The thumbprint can be pasted as the certificate dialog shows it, with spaces. The certificate must be in *Local Computer → Personal* and have a private key; the script warns if `-HostName` is not one of its DNS names.
   - The service account gets *Log on as a service* on this server. If a domain GPO manages that right, add the account there too, or the next policy refresh removes it.

4. Fill in `C:\ProgramData\EndpointManagementAPI\config\appsettings.json` and `secrets.json` ([06](06-configuration.md)), including `entra.adminUiClientId` for the admin UI ([03, part C](03-entra-setup.md)). Write the secrets in plain text – the service encrypts them with DPAPI for its own account at the first start ([06](06-configuration.md#encryption-with-dpapi)).
5. `Start-Service EndpointManagementAPI`

What the script does: checks the release folder (above), copies the release to `C:\Program Files\EndpointManagementAPI` (previous version kept as `...\EndpointManagementAPI.previous`; owner Administrators, only administrators and SYSTEM may change it, everyone else reads), sets owner and access of `C:\ProgramData\EndpointManagementAPI` (also when the folder existed before: owner Administrators on every file and folder, the access of everything below reset, so no access entry someone added before the install stays; administrators and SYSTEM full, the service account read; nothing inherited from `ProgramData`; it stops if the folder or anything in it is a junction or symbolic link), creates `config` and `logs` with access for administrators, SYSTEM and the service account only and `pending-primary-users` writable for the service account, registers the service, sets its account, and configures IIS (`Configure-IISProxy.ps1`: URL Rewrite rule, ARR proxy, *Require SSL* on the site or application so IIS refuses plain HTTP, and `X-Forwarded-Proto` set from how the client really came in). A first rule answers 400 to a path with an encoded `?` or `#` (`%3F`, `%23`): IIS would decode it before forwarding, and the service would act on a shorter name than the caller sent.

## First checks

| Check | Expected |
|---|---|
| `Get-Service EndpointManagementAPI` | Running |
| `Invoke-RestMethod https://<host>/<path>/api/admin/ui-config` | `success: true` with tenant and client ID |
| Same URL `/api/monitoring/health` without token | HTTP 401 – the API is up and checks tokens |
| Browser `https://<host>/<path>/` | admin UI sign-in page |
| Admin UI → *Status* | every connection "Connected" |
| PowerShell module with a client app registration: `Test-ConfigMgrDevice -ComputerName <known PC>` | `success: True, data: True` |

## Reaching it from outside the network (optional)

Without VPN and without an inbound port, the admin UI and the API can be published through [Microsoft Entra application proxy](https://learn.microsoft.com/en-us/entra/identity/app-proxy/overview-what-is-app-proxy). A connector on a Windows server in the network opens only outbound connections to Microsoft (ports 80 and 443); IIS and the service stay as they are. This setup is not tested in the lab yet.

| Setting of the application proxy app | Value |
|---|---|
| Internal URL | the address IIS answers on, e.g. `https://endpoint-management.contoso.local/EndpointManagementAPI/`; the connector server must trust its certificate |
| External URL | the same path, e.g. `https://endpoint-management-contoso.msappproxy.net/EndpointManagementAPI/`, or your own domain with its certificate uploaded to the app |
| Pre-authentication | **Passthrough** (reason below) |
| Translate URLs in headers | Yes (default) |
| Connector | at least two Windows servers with line of sight to the API server, for availability |

Then:

1. Add the external address as a further single-page application redirect URI of `EndpointManagement.AdminUI` ([03, part C](03-entra-setup.md#c-endpointmanagementadminui--the-ui-registration-required-for-the-ui)), with the trailing `/`.
2. Put Conditional Access on `EndpointManagement.AdminUI`, e.g. multifactor authentication and a compliant device. It applies at sign-in, so it covers the external address too.
3. Clients that should use the external address set it as `BaseUrl`; everything else stays the same.

**Why Passthrough:** the service checks every Entra ID token itself, as it does inside the network. Pre-authentication *Microsoft Entra ID* would put a second sign-in for the proxy app in front of it, which client systems with client credentials and task sequences in WinPE cannot do. If only people use the external address, it can be tried as an extra layer.

**Licence and the alternative:** application proxy is part of Microsoft Entra ID P1. [Microsoft Entra Private Access](https://learn.microsoft.com/en-us/entra/architecture/sse-deployment-guide-private-access) uses the same connector and reaches the internal address directly, but needs its own licence and the Global Secure Access client on the device; task sequences in WinPE have no such client and stay in the network.

## Update

Run `Install-EndpointManagementApi.ps1` from the new release with the same parameters. It stops the service, keeps the old version as `.previous`, copies the new one, refreshes the service definition and starts it. Configuration in `ProgramData` is never overwritten; new `*.example.json` files show new settings. The installer also creates the Windows Event Log *Endpoint Management API* if it is missing.

## Rollback

```powershell
Stop-Service EndpointManagementAPI
Remove-Item 'C:\Program Files\EndpointManagementAPI' -Recurse -Force
Rename-Item 'C:\Program Files\EndpointManagementAPI.previous' 'EndpointManagementAPI'
Start-Service EndpointManagementAPI
```

## Uninstall

`.\deploy\windows\Uninstall-EndpointManagementApi.ps1` removes the service and the IIS application. Add `-RemoveData` to delete configuration and logs as well.

## Building the release yourself

On a Windows machine with Node.js 24:

```powershell
npm ci; npm test; npm run build
$env:WINSW_EXE = 'C:\Tools\WinSW.NET461.exe'       # WinSW 2.12, https://github.com/winsw/winsw/releases/tag/v2.12.0
$env:NODE_RUNTIME_DIR = 'C:\Tools\node-v24-win-x64'  # optional, default: the node.exe that runs the script
node scripts/package.mjs                              # → out\EndpointManagementAPI-<version>.zip
```

Build on Windows: the optional `kerberos` module is native and must match the target OS.

What `package.mjs` guarantees:

- The production `node_modules` come from `package-lock.json` with `npm ci --omit=dev --ignore-scripts`, the tree the tests ran against, and every package is checked against the lockfile; a version that is not locked stops the build. No install script of a package runs.
- `kerberos` is compiled from its locked source with node-gyp (needs the Visual Studio build tools), never with its install script, which would download a prebuilt binary. Without a compiler it is left out: only `configMgr.transport` `node` needs it.
- `WinSW.exe` must have the SHA-256 of WinSW.NET461.exe 2.12.0, otherwise the build stops.
- Only `*.example.json` from `apps/api/config` go into the release, never your own settings, secrets, sequences or learned host names.
- `release-manifest.json` lists every file with its SHA-256; its own SHA-256 is printed and written to the GitHub job summary.

The GitHub workflow runs with a read-only token, does not keep it after checkout, uses the actions and the Node.js version pinned to exact versions, and attests `release-manifest.json` where GitHub supports it.
