---
title: Server.Slack.Clients.Online
hidden: true
tags: [Server Event Artifact]
---

Send a message to slack when clients come online.

This artifact searches for all clients that carry the label "Slack"
by default, and if they have appeared online in the last 5 minutes,
sends a message to Slack and removed the label from the client.


```yaml
name: Server.Slack.Clients.Online
description: |
   Send a message to slack when clients come online.

   This artifact searches for all clients that carry the label "Slack"
   by default, and if they have appeared online in the last 5 minutes,
   sends a message to Slack and removed the label from the client.

type: SERVER_EVENT

parameters:
  - name: LabelGroup
    default: Slack
  - name: SlackToken
    description: The token URL obtained from Slack. Leave blank to use server metadata. e.g. https://hooks.slack.com/services/XXXX/YYYY/ZZZZ

sources:
  - query: |
        LET token_url = if(
           condition=SlackToken,
           then=SlackToken,
           else=server_metadata().SlackToken)

        LET hits = SELECT client_id,
               os_info.fqdn as Hostname ,
               now() - last_seen_at / 1000000 AS LastSeen,
               label(client_id=client_id, labels=LabelGroup, op="remove")
        FROM clients(search="label:" + LabelGroup)
        WHERE LastSeen < 300

        LET send_massage = SELECT * FROM foreach(row=hits,
        query={
           SELECT client_id, Hostname, LastSeen, Content, Response
           FROM http_client(
                data=serialize(item=dict(
                text=format(format="Client %v (%v) has appeared online %v seconds ago",
                            args=[Hostname, client_id, LastSeen])),
                format="json"),
            headers=dict(`Content-Type`="application/json"),
            method="POST",
            url=token_url)
        })

        // Check every minute
        SELECT * FROM foreach(
           row={SELECT * FROM clock(period=60)},
           query=send_massage)

```
