---
title: Server.Monitor.Health
hidden: true
tags: [Server Event Artifact]
---

This is the main server health dashboard. It is shown on the
homescreen and enabled by default on all new installs.

You may edit this artifact to customize your server dashboard.

Alternatively, edit the Welcome screen at the
`Server.Internal.Welcome` artifact.


<pre><code class="language-yaml">
name: Server.Monitor.Health
description: |
  This is the main server health dashboard. It is shown on the
  homescreen and enabled by default on all new installs.

  You may edit this artifact to customize your server dashboard.

  Alternatively, edit the Welcome screen at the
  `Server.Internal.Welcome` artifact.

type: SERVER_EVENT

sources:
  - name: Prometheus
    query: SELECT sleep(time=10000000) FROM scope()

reports:
  - type: SERVER_EVENT
    # Only allow the report to run for 10 seconds - this is plenty for
    # the GUI.
    timeout: 10
    parameters:
      - name: Sample
        default: "6"

    template: |
      {{ define "CPU" }}
          SELECT _ts as Timestamp,
              CPUPercent,
              int(int=MemoryUse / 1048576) AS MemoryUse_Mb,
              TotalFrontends
          FROM source(source="Prometheus",
                      start_time=StartTime, end_time=EndTime,
                      artifact="Server.Monitor.Health")
      {{ end }}

      {{ define "CurrentConnections" }}
           SELECT * FROM sample(
             n=atoi(string=Sample),
             query={
               SELECT _ts as Timestamp,
                  client_comms_current_connections
               FROM source(source="Prometheus",
                           start_time=StartTime, end_time=EndTime,
                           artifact="Server.Monitor.Health")
            })
      {{ end }}

      {{ $time_rows := Query "SELECT timestamp(epoch=now()) AS Now FROM scope()" | Expand }}
      {{ $time := Get $time_rows "0.Now" }}
      ## Server status @ &lt;velo-value value="{{ $time.Format "2006-01-02T15:04:05Z07:00" }}" /&gt;

      &lt;p&gt;The following are total across all frontends.&lt;/p&gt;
          &lt;span class="container"&gt;
            &lt;span class="row"&gt;
              &lt;span class="col-sm panel"&gt;
               CPU and Memory Utilization
               {{- Query "CPU" | LineChart "xaxis_mode" "time" "RSS.yaxis" 2 -}}
              &lt;/span&gt;
              &lt;span class="col-sm panel"&gt;
               Currently Connected Clients
               {{- Query "CurrentConnections" | LineChart "xaxis_mode" "time" "RSS.yaxis" 2 -}}
              &lt;/span&gt;
            &lt;/span&gt;
      &lt;/span&gt;

      ## Current Orgs

      {{ Query "LET ColumnTypes &lt;= dict(ClientConfig='url_internal') \
                SELECT Name, OrgId, \
                       format(format='[%s](/notebooks/Dashboards/%s/uploads/data/client.%s.config.yaml)', \
                       args=[OrgId, ArtifactName, OrgId]) AS ClientConfig, \
                       upload(accessor='data', file=_client_config, \
                              name='client.'+OrgId+'.config.yaml') AS _Upload \
                FROM orgs() " | Table }}

      ## Disk Space

      {{ Query "SELECT * FROM Artifact.Generic.Client.DiskSpace()" | Table }}

      ## Users

      {{ define "UserPermissions" }}
        SELECT name, effective_policy AS _EffectivePolicy,
               join(array=roles, sep=", ") AS Roles
        FROM gui_users()
      {{ end }}

      {{ Query "UserPermissions" | Table }}

      ## Server version

      {{ Query "SELECT server_version FROM config" | Table }}

</code></pre>

