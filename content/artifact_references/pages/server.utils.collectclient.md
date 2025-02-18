---
title: Server.Utils.CollectClient
hidden: true
tags: [Server Artifact]
---

This artifact simplifies collecting from a specific client by
performing all steps automatically:

1. The collection will be scheduled.
2. The artifact will wait for the collection to complete
3. The results from the collection will be displayed.


```yaml
name: Server.Utils.CollectClient
description: |
  This artifact simplifies collecting from a specific client by
  performing all steps automatically:

  1. The collection will be scheduled.
  2. The artifact will wait for the collection to complete
  3. The results from the collection will be displayed.

type: SERVER

parameters:
  - name: ArtifactName
    default: Generic.Client.Info
  - name: Client
    description: A client ID or a Hostname
    default: C.1234
  - name: Parameters
    default: "{}"
    description: A key/value JSON object specifying parameters for the artifact
    type: json

sources:
  - query: |
      -- Find a client to collect from by applying the search
      -- critertia and picking the first hit
      LET clients = SELECT client_id FROM clients(search=Client) LIMIT 1
      LET client_id <= clients[0].client_id

      -- If we found something then schedule the collection.
      LET collection <= if(condition=client_id,
          then=collect_client(client_id=client_id,
                              artifacts=ArtifactName, env=Parameters),
          else=log(message="No clients found to match search " + Client) AND FALSE)

      -- Wait for the collection to finish - if the client is
      -- currently connected this wont take long
      LET flow_results <= SELECT * FROM if(condition=collection,
      then={
         SELECT * FROM watch_monitoring(artifact='System.Flow.Completion')
         WHERE FlowId = collection.flow_id
         LIMIT 1
      })

      -- Collect the results
      SELECT * FROM foreach(row=flow_results[0].Flow.artifacts_with_results,
      query={
           SELECT *, _value AS Source, client_id
           FROM source(client_id=client_id, flow_id=collection.flow_id, artifact=_value)
      })

```
