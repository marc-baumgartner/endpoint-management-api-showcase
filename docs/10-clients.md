# 10 – Clients

A client is any system that calls the API without a person in front of it: OSD task sequences, ServiceNow or another ITSM tool, scheduled scripts, monitoring. Every client

1. has **its own app registration** with only the roles it needs ([03, part E](03-entra-setup.md#e-client-app-registrations--one-per-client-system)),
2. gets an access token from Entra ID with the **client credentials** flow,
3. calls the API over HTTPS with `Authorization: Bearer <token>`.

How it sends the calls is up to the client:

| | PowerShell module | REST directly |
|---|---|---|
| For | task sequences, WinPE, Windows scripts | ServiceNow and other ITSM tools, Linux, anything with an HTTP client |
| Token | handled by `Initialize-Api` (secret or certificate), cached and renewed | the client requests it itself (below) |
| Calls | one cmdlet per route, e.g. `Add-ConfigMgrDeviceToCollection` | `POST https://<host>/<path>/api/configmgr/devices/PC01/collections/Apps%20-%20Office` |
| Answer | the envelope as object (`.data` holds the answer), errors as PowerShell errors | the envelope `{ success, data, error, timestamp }` as JSON |
| Docs | [08](08-powershell-module.md) | [09](09-api-reference.md), OpenAPI at `/api/docs` |

Both use the same token, routes and roles – the module is only a convenience layer.

## Values a client needs

| Value | Example |
|---|---|
| Base URL | `https://sccm01.contoso.local/EndpointManagementAPI` |
| Tenant ID | `11111111-1111-1111-1111-111111111111` |
| Client ID | of **its own** registration, e.g. `EndpointManagement.Client.ServiceNow` |
| Credential | certificate (private key on the client) or client secret |
| Scope | `api://<client ID of EndpointManagement.Service>/.default` |

## Getting a token (REST)

`POST https://login.microsoftonline.com/<tenant id>/oauth2/v2.0/token`, form-encoded:

| Field | Value |
|---|---|
| `grant_type` | `client_credentials` |
| `client_id` | the client's own client ID |
| `client_secret` | the secret – **or** `client_assertion_type` = `urn:ietf:params:oauth:client-assertion-type:jwt-bearer` and `client_assertion` = a JWT signed with the certificate |
| `scope` | `api://<Service client ID>/.default` |

The answer holds `access_token` and `expires_in` (seconds, usually 3600–5400). Reuse the token until shortly before it expires; do not request one per call.

The token's `roles` claim lists the app roles granted to the client. If it is missing, admin consent was not granted ([03, part E](03-entra-setup.md#e-client-app-registrations--one-per-client-system), step 3).

## Examples

### PowerShell without the module

```powershell
$tenant  = '11111111-1111-1111-1111-111111111111'
$service = '00000000-0000-0000-0000-000000000000'          # EndpointManagement.Service
$base    = 'https://sccm01.contoso.local/EndpointManagementAPI'

$token = (Invoke-RestMethod -Method Post "https://login.microsoftonline.com/$tenant/oauth2/v2.0/token" -Body @{
    grant_type    = 'client_credentials'
    client_id     = '<client id>'
    client_secret = $secret
    scope         = "api://$service/.default"
}).access_token
$headers = @{ Authorization = "Bearer $token" }

(Invoke-RestMethod "$base/api/configmgr/devices/PC01/exists" -Headers $headers).data      # True / False

Invoke-RestMethod -Method Post "$base/api/configmgr/devices/PC01/status-messages" -Headers $headers `
    -ContentType 'application/json' -Body (@{ Severity = 'Information'; Component = 'OSD'; Description = 'Apply OS done' } | ConvertTo-Json)
```

### curl

```bash
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/$TENANT/oauth2/v2.0/token" \
  -d grant_type=client_credentials -d client_id="$CLIENT_ID" --data-urlencode client_secret="$SECRET" \
  -d scope="api://$SERVICE_APP_ID/.default" | jq -r .access_token)

curl -s -H "Authorization: Bearer $TOKEN" "$BASE/api/intune/devices/PC01/compliance"
# {"success":true,"data":{"complianceState":"compliant","lastSyncDateTime":"2026-11-05T08:00:00Z"},"timestamp":"2026-11-06T14:30:00Z"}

curl -s -X PUT -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"TargetOU":"OU=Clients,OU=Computers,DC=contoso,DC=local"}' "$BASE/api/ad/computers/PC01/ou"
```

### ServiceNow

Set up once (names of the menus can differ slightly between ServiceNow releases):

1. **Credential** – *System OAuth → Application Registry → New → Connect to a third party OAuth Provider*:
   - Client ID / Client Secret: from `EndpointManagement.Client.ServiceNow`
   - Default Grant type: **Client Credentials**
   - Token URL: `https://login.microsoftonline.com/<tenant id>/oauth2/v2.0/token`
   - OAuth Entity Scope: `api://<Service client ID>/.default` (also add it to the OAuth Entity Profile)
2. **REST Message** – *System Web Services → Outbound → REST Message → New*:
   - Endpoint: `https://<host>/<path>/api/...`
   - Authentication type: **OAuth 2.0**, OAuth profile from step 1
   - One HTTP method per route you need, e.g. `POST …/api/intune/devices/${computerName}/sync`, with the header `Content-Type: application/json` for bodies.
3. In a workflow or script, check `success` in the answer and show `error` to the agent when it is `false`.

ServiceNow requests and renews the token itself. The API must be reachable from ServiceNow – through a MID Server, if the API is only reachable internally (set the MID Server on the REST Message).

### Monitoring probe

A probe needs only `Monitoring.Read`. `GET /api/monitoring/health` answers **200** when every enabled connection works and **503** otherwise, with the reason per connection in `checks` ([07](07-operations.md#health)).

## Request bodies

Bodies are JSON; property names are matched ignoring case.

| Route | Body |
|---|---|
| `POST /api/configmgr/devices` | `{ "ComputerName": "PC01", "BiosGuid": "4C4C4544-…" }` |
| `POST …/configmgr/devices/{computerName}/primary-users` | `{ "UserName": "jdoe" }` |
| `PUT …/configmgr/devices/{computerName}/primary-users` | `{ "UserList": ["jdoe", "asmith"] }` – at most 100 names; letters, digits, `.`, `_`, `-`, `@` and `\`, up to 104 characters each |
| `POST …/osd-collections`, `POST …/restaging` | `{ "Identifier": "24H2" }` (key from `configMgr.osDeployments`) |
| `PUT …/variables/{variableName}` | `{ "Value": "de-CH" }` |
| `PUT …/variables` | `{ "Variables": [ { "Key": "OSDLanguage", "Value": "de-CH" } ] }` – replaces all variables; reserved variables on the device stay |
| `POST …/status-messages` | `{ "Severity": "Information" \| "Warning" \| "Error", "Component": "OSD", "Description": "…" }` – `Component` may not start with `SMS_` or `CCM` |
| `POST /api/configmgr/usmt` | `{ "SourceComputerName": "OLD01", "DestinationComputerName": "NEW01" }` |
| `POST …/policy-reset` | `{ "FullReset": false }` (body optional) |
| `PUT /api/ad/computers/{computerName}/ou` | `{ "TargetOU": "OU=Clients,DC=contoso,DC=local" }` |
| `PUT /api/ad/computers/{computerName}/description` | `{ "Description": "…" }` – empty clears it |
| `PUT /api/intune/devices/{computerName}/primary-user` | `{ "UserName": "jdoe" }` |
| `PUT /api/intune/devices/{computerName}/category` | `{ "CategoryName": "Kiosk" }` |

Path parameters with spaces or special characters are URL-encoded (`Apps%20-%20Office`).

**Device variables:** reserved names answer 403 (`API_*`, `_SMSTS*`, `SMSTSPostAction`, `OSDJoin*` …, list in [06](06-configuration.md#scope-what-a-readwrite-role-may-reach)): they run commands, carry credentials or the module's sign-in. An existing variable keeps its *masked* flag when its value changes. An empty value is written as one space (ConfigMgr stores no empty values), so the device variable still overrides a collection variable of the same name. When the AdminService returns masked variables of the device without their value, every variable change answers 409, because writing the list back would empty them; change those devices in the console.

## Good practice

- **One registration per client system**, never shared. The audit log (admin UI → *Audit*) shows the app name of every call.
- **Least privilege:** Read roles for clients that only look things up; ReadWrite only where the client changes something. `Intune.ReadWrite` includes retire and delete but not wipe (`Intune.Wipe`); BitLocker recovery keys need `AD.BitLocker.Read`.
- **Rate limits:** each client has its own limits (default 300 reads and 60 changes per minute, 20 critical actions per hour). On **429** wait the seconds in `Retry-After` and try again; a TaskSequence client needs nothing special, a bulk job should pace itself.
- **Certificates** where the client can hold a private key; **secrets** only with a short lifetime and an owner who renews them.
- **Retry** only on 503 and network errors, with a pause. 4xx answers do not change on retry.
- **Do not parse messages.** Use the HTTP status and `success`; `error` is for people.
