---
title: Windows.Forensics.FilenameSearch
hidden: true
tags: [Client Artifact]
---

Did a specific file exist on this machine in the past or does it
still exist on this machine?

This common question comes up frequently in cases of IP theft,
discovery and other matters. One way to answer this question is to
search the $MFT file for any references to the specific filename. If
the filename is fairly unique then a positive hit on that name
generally means the file was present.

Simply determining that a filename existed on an endpoint in the
past is significant for some investigations.

This artifact applies a YARA search for a set of filenames of
interest on the $MFT file. For any hit, the artifact then identified
the MFT entry where the hit was found and attempts to resolve that
to an actual filename.


```yaml
name: Windows.Forensics.FilenameSearch
description: |
  Did a specific file exist on this machine in the past or does it
  still exist on this machine?

  This common question comes up frequently in cases of IP theft,
  discovery and other matters. One way to answer this question is to
  search the $MFT file for any references to the specific filename. If
  the filename is fairly unique then a positive hit on that name
  generally means the file was present.

  Simply determining that a filename existed on an endpoint in the
  past is significant for some investigations.

  This artifact applies a YARA search for a set of filenames of
  interest on the $MFT file. For any hit, the artifact then identified
  the MFT entry where the hit was found and attempts to resolve that
  to an actual filename.

parameters:
    - name: yaraRule
      default: |
        rule Hit {
           strings:
             $a = "my secret file.txt" nocase wide ascii
           condition:
             any of them
        }
      type: yara
    - name: Device
      default: "C:"

sources:
  - query: |
        SELECT String.Offset AS Offset,
               String.HexData AS HexData,
               parse_ntfs(device=Device,
                          mft=String.Offset / 1024) AS MFT
        FROM yara(
             rules=yaraRule, files=Device + "/$MFT",
             end=10000000000,
             number=1000,
             accessor="ntfs")

```
