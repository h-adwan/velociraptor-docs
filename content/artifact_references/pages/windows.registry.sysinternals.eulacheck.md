---
title: Windows.Registry.Sysinternals.Eulacheck
hidden: true
tags: [Client Artifact]
---

Checks for the Accepted Sysinternals EULA from the registry key
"HKCU\Software\Sysinternals\[TOOL]\".  When a Sysinternals tool is
first run on a system, the EULA must be accepted. This writes a
value called EulaAccepted under that key.

Note: This artifact uses HKEY_USERS and therefore will not detect
users that are not currently logged on.


```yaml
name: Windows.Registry.Sysinternals.Eulacheck
description: |
  Checks for the Accepted Sysinternals EULA from the registry key
  "HKCU\Software\Sysinternals\[TOOL]\".  When a Sysinternals tool is
  first run on a system, the EULA must be accepted. This writes a
  value called EulaAccepted under that key.

  Note: This artifact uses HKEY_USERS and therefore will not detect
  users that are not currently logged on.

parameters:
   - name: Sysinternals_Reg_Key
     default: HKEY_USERS\*\Software\Sysinternals\*
   - name: userRegex
     default: .
     type: regex

imports:
   - Windows.Registry.NTUser

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    name: RegistryAPI
    query: |
      LET users <= SELECT Name, UUID
          FROM Artifact.Windows.Sys.Users()
      WHERE Name =~ userRegex

      SELECT Key.Name as ProgramName,
             Key.FullPath as Key,
             Key.Mtime AS TimeAccepted,
             {
                SELECT Name FROM users WHERE UUID=regex_replace(
                   source=Key.FullPath, re=".+\\\\(S-[^\\\\]+)\\\\.+", replace="$1")
             } as User,
             EulaAccepted
      FROM read_reg_key(globs=split(string=Sysinternals_Reg_Key, sep=',[\\s]*'))

  - name: RawRegistry
    description: Detect keys using Raw Registry Analysis
    query: |
      -- Apply Raw Registry Mappings
      LET _ <= MapRawRegistryHives

      -- Make sure to call the other sources otherwise we get recursion errors!
      SELECT *
      FROM Artifact.Windows.Registry.Sysinternals.Eulacheck(source="RegistryAPI")

```
