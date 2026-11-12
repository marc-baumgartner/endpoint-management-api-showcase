# 05 – Active Directory

The service talks **LDAPS** directly to the domain controllers with a dedicated service account (`ldapts`, simple bind, TLS only).

## Service account

Create `svc-emapi-ldap` (not a gMSA – a simple bind needs a password), password never expires, rotated by your process, stored in `secrets.json` or `EMAPI_LDAP_BIND_PASSWORD`.

Delegate on the OUs the API manages (*Delegate Control* or `dsacls`):

| Right | Object | For |
|---|---|---|
| Read all properties | computers, users, groups | info, exists, group lists |
| Write `description` | computers | `PUT …/description` |
| Write `member` | the groups the API may change (e.g. all `G_SCCM_*` groups in one OU) | add/remove group members |
| Create/delete computer objects, *Delete subtree*, move | source and target OUs | `PUT …/ou`, `DELETE /computers/…` (tree delete removes BitLocker child objects too) |
| Read `msFVE-RecoveryPassword` | `msFVE-RecoveryInformation` below computers | BitLocker routes – grant only if needed |
| Write `userAccountControl` | computers | `PUT …/enabled` (enable/disable the account) |
| Write `info`, `location`, `extensionAttribute1`–`15` | computers | `PUT …/attributes/{attributeName}` – grant only the attributes you use |
| Create computer objects | the OUs new computers are created in | `POST /api/ad/computers` (prestage) |
| Read `msLAPS-Password`, `msLAPS-PasswordExpirationTime` (Windows LAPS) or `ms-Mcs-AdmPwd` (legacy LAPS) | computers | `GET …/laps` – grant only if needed, e.g. with `Set-LapsADReadPasswordPermission` |

### Commands

As a domain admin, on a DC or with RSAT. `$computers` are the OUs with the computers the API manages (including target OUs for moves), `$groups` the OUs with the groups whose members it may change. Leave out lines for features you do not use.

```powershell
$acct      = 'CONTOSO\svc-emapi-ldap'
$computers = @('OU=Clients,DC=contoso,DC=local')
$groups    = @('OU=Groups,DC=contoso,DC=local')

foreach ($ou in $computers) {
    dsacls $ou /I:T /G "${acct}:GR"                             # read everything
    dsacls $ou /I:T /G "${acct}:CCDC;computer"                  # create and delete computers
    dsacls $ou /I:S /G "${acct}:SDDT;;computer"                 # delete including BitLocker child objects
    foreach ($attr in 'description','userAccountControl','info','location','cn','name') {
        dsacls $ou /I:S /G "${acct}:WP;$attr;computer"          # write; cn and name are needed to move
    }
    1..15 | ForEach-Object { dsacls $ou /I:S /G "${acct}:WP;extensionAttribute$_;computer" }
    dsacls $ou /I:S /G "${acct}:RPCA;msFVE-RecoveryPassword;msFVE-RecoveryInformation"   # BitLocker keys
    Set-LapsADReadPasswordPermission -Identity $ou -AllowedPrincipals $acct               # Windows LAPS, not encrypted
}
foreach ($ou in $groups) {
    dsacls $ou /I:T /G "${acct}:GR"
    dsacls $ou /I:S /G "${acct}:WP;member;group"                # change group members
}
```

The account itself: `New-ADUser -Name svc-emapi-ldap -Path '<OU for service accounts>' -AccountPassword (Read-Host -AsSecureString) -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true`. Encrypted Windows LAPS is read by the **service account** (gMSA), not by this account, see below.

`Set-LapsADReadPasswordPermission` needs the `LAPS` module (Windows Server 2019 or later with the April 2023 update). Without it, the same right with dsacls: `dsacls $ou /I:S /G "${acct}:RPCA;msLAPS-Password;computer"`; for legacy LAPS `Set-AdmPwdReadPasswordPermission` (module AdmPwd.PS) or `RPCA;ms-Mcs-AdmPwd;computer`.

A missing right shows as 403 with the attribute in the message; add only that line. In a lab one parent OU for both lists is enough; in production keep them to the client OUs and the one group OU, so the account cannot delete servers or change other groups.

## Connection

```json
"activeDirectory": {
  "urls": ["ldaps://dc01.contoso.local:636", "ldaps://dc02.contoso.local:636"],
  "baseDn": "DC=contoso,DC=local",
  "bindDn": "CN=svc-emapi-ldap,OU=Service Accounts,DC=contoso,DC=local"
}
```

- URLs are tried in order; the second one is used when the first does not answer.
- The DC certificates must be trusted. The service starts Node.js with `--use-system-ca`, so it trusts every CA in the Windows certificate store of the server (*Trusted Root Certification Authorities*), like any Windows program. An internal CA distributed by GPO therefore needs no extra step. The same applies to the AdminService certificate.
- Every search value is escaped (RFC 4515), DN values per RFC 4514. Route parameters are additionally validated (allowed characters, length).

## Operations

| Route | LDAP | Notes |
|---|---|---|
| find a computer (every route) | `(&(objectClass=computer)(\|(sAMAccountName=<name>$)(cn=<name>)))` | one search for both names. One hit is used; with several, the one whose `cn` **and** `sAMAccountName` match wins, otherwise 409 instead of using one of them (a long name cut to 15 characters in `sAMAccountName` cannot stand in for another computer) |
| computer exists / info | as above | `ouPath` top-level first; `lastLogon` ISO 8601 UTC (value of the answering DC) |
| delete computer | delete with the *Tree Delete* control | also removes child objects (BitLocker recovery information); needs *Delete subtree* |
| move to OU | ModifyDN to `TargetOU` (a DN) | 404 when the computer or the OU does not exist; 400 when `TargetOU` is not an OU or container; 403 for the Domain Controllers OU (and every other critical system OU) and for OUs outside `managedOus`. The checks use the DN as AD returns it, so other spellings (`\20`, escaped letters, spaces around `=`) are caught too |
| set description | replace / delete `description` | an empty or missing value clears it |
| computer groups, user groups | `tokenGroups` of the object, resolved to `sAMAccountName` | includes nested groups and the primary group |
| add / remove computer or user to group | `member` add/delete on the group found by `sAMAccountName`, else by `cn` | answers `Successful`, `AlreadyMember` or `NotMember`; the membership check also works for groups with more than 1500 members. 403 for protected groups and groups outside `managedGroups` (see below) |
| user exists / info | `(&(objectCategory=person)(objectClass=user)(sAMAccountName=…))` | a `DOMAIN\` prefix is ignored; `enabled` from `userAccountControl` |
| BitLocker key IDs / recovery key | `msFVE-RecoveryInformation` one level below the computer | key ID from `msFVE-RecoveryGuid`; a request may list several IDs, with or without braces; an ID matches the whole key ID or its start with at least 8 hexadecimal characters (as the recovery screen shows it), never a shorter fragment |
| enable / disable computer | replace `userAccountControl` with the `ACCOUNTDISABLE` bit (2) set or cleared | every other flag stays as it is |
| set computer attribute | replace / delete the attribute | allowed: `info`, `location`, `extensionAttribute1`–`15`; any other name is 400. An empty value clears it |
| create (prestage) computer | add `CN=<name>,<TargetOU>` with `objectClass` computer, `sAMAccountName` `<NAME>$` and, with `prestageAccount` `randomPassword` (default), `userAccountControl` 4096 (workstation trust) plus a 68-character random `unicodePwd` that is not kept anywhere; with `disabled`, `userAccountControl` 4098 and no password | 409 when the name exists anywhere in the domain, 404 when the OU does not exist, 403 outside `managedOus` and for the Domain Controllers OU. The device joins with this name by an account allowed to reset its password (task sequence join account, `djoin`); the join sets a new password and enables the account. The account never exists enabled without a password, unlike one created in *Active Directory Users and Computers* (4128, password not required). Setting a password needs LDAPS |
| LAPS password | Windows LAPS `msLAPS-Password`, else `msLAPS-EncryptedPassword`, else legacy `ms-Mcs-AdmPwd` | see below |

### What the API may change

- **Never:** domain controllers, read-only DCs and (group) managed service accounts (`userAccountControl` server trust, `primaryGroupID` 516/521, `msDS-(Group)ManagedServiceAccount`, `isCriticalSystemObject`). Every change and every LAPS or BitLocker read answers 403; reading the info still works.
- **Never:** groups in `activeDirectory.protectedGroups` (default `G_EMAPI_*`), the built-in administrative groups (whatever the list says, DnsAdmins, Group Policy Creator Owners and Cert Publishers included), groups with `adminCount=1`, built-in groups, and every group nested in one of these on any level (search with `member:1.2.840.113556.1.4.1941:=`; a member of the inner group would get the rights of the outer one).
- **Only, when set:** computers in `activeDirectory.managedOus` and groups in `activeDirectory.managedGroups` ([06](06-configuration.md#scope-what-a-readwrite-role-may-reach)).

These checks are in the service; the delegation above stays the second line. Delegate only the OUs in `managedOus`, and never the OU of the `G_EMAPI_*` groups if they are synchronised from AD.

## LAPS passwords

`GET /api/ad/computers/{computerName}/laps` needs the role `LAPS.Read` and answers `{ accountName, password, expiresAt, backupDateTime, source }`.

`GET /api/ad/computers/{computerName}/laps/status` only says whether a password is stored, from which LAPS and when it changes, from the expiration time, which is not confidential. It needs `AD.Read` and is not a critical action. The device page uses it (and `GET /api/entra/devices/{computerName}/laps/status` for Entra ID) to grey out *Show Local Admin Password* and *Rotate Local Admin Password* when no password is stored.

- **Windows LAPS, not encrypted** (`msLAPS-Password`): read over LDAP; the bind account needs read access to the attribute.
- **Windows LAPS, encrypted** (`msLAPS-EncryptedPassword`, the default on 2016+ domain functional level): LDAP only returns the encrypted blob. The service decrypts it through its PowerShell helper with `Get-LapsADPassword -AsPlainText`, running as the **service account (gMSA)**. That account must be an authorized decryptor: put it into the group named in the LAPS policy *Configure authorized password decryptors* (default: Domain Admins – use a dedicated group instead) and give it read access with `Set-LapsADReadPasswordPermission -Identity <OU> -AllowedPrincipals <gMSA>`. The RSAT module `LAPS` ships with Windows Server 2019+ (April 2023 update). Without the helper (e.g. on Linux) the route answers 501.
- **Legacy LAPS** (`ms-Mcs-AdmPwd`): read over LDAP, `accountName` is empty (legacy LAPS does not store it).

No password stored: 404. Every call is a critical action: Event Log 3100 and the hourly critical limit ([07](07-operations.md)).

An unknown computer, user or group answers **404** with a message. Exceptions: `GET …/computers/{name}/groups` and the BitLocker key IDs of an unknown computer answer an empty list.

The DDR after `RegisterDevice` reads SID, OUs and direct groups of the computer from here as well ([04](04-configmgr-adminservice.md#windows-helper-process)).
