---
title: Server.Alerts.WinPmem
hidden: true
tags: [Server Event Artifact]
---

Send an email if the pmem service has been installed on any of the
endpoints.

Note this requires that the Windows.Event.ServiceCreation
monitoring artifact be collected from clients.


```yaml
name: Server.Alerts.WinPmem
description: |
   Send an email if the pmem service has been installed on any of the
   endpoints.

   Note this requires that the Windows.Event.ServiceCreation
   monitoring artifact be collected from clients.

type: SERVER_EVENT

parameters:
  - name: EmailAddress
    default: admin@example.com
  - name: SkipVerify
    type: bool
    description: If set we skip TLS verification.

sources:
  - query: |
        SELECT * FROM foreach(
          row={
            SELECT * from watch_monitoring(
              artifact='Windows.Events.ServiceCreation')
            WHERE ServiceName =~ 'pmem'
          },
          query={
            SELECT * FROM mail(
              to=EmailAddress,
              subject='Pmem launched on host',
              period=60,
              skip_verify=SkipVerify,
              body=format(
                 format="WinPmem execution detected at %s for client %v",
                 args=[Timestamp, ClientId]
              )
          )
        })

```
