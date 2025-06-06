---
title: Admin.Client.UpdateClientConfig
hidden: true
tags: [Client Artifact]
---

Sometimes we wish to move a client from one org ID to another. This
requires updating the config on the client and rekeying the client.

This artifact will replace the client's config file and restart
it. The config file will be verified before replacing it. If set to
not rekey, the client will retain its client id but will be killed -
the service manager should restart it and cause the new config to
reload.

This artifact has a notebook suggestion that allows a client to be
changed to a different org.


<pre><code class="language-yaml">
name: Admin.Client.UpdateClientConfig
description: |
  Sometimes we wish to move a client from one org ID to another. This
  requires updating the config on the client and rekeying the client.

  This artifact will replace the client's config file and restart
  it. The config file will be verified before replacing it. If set to
  not rekey, the client will retain its client id but will be killed -
  the service manager should restart it and cause the new config to
  reload.

  This artifact has a notebook suggestion that allows a client to be
  changed to a different org.

parameters:
   - name: ConfigYaml
     description: The new config to write in yaml form.
   - name: ConfigPath
     description: Path of config file to overwrite
   - name: WaitPeriod
     type: int
     default: 10
   - name: RekeyClient
     type: bool
     default: Y
     description: Should the client rekey its client ID.

required_permissions:
  - EXECVE
  - FILESYSTEM_WRITE

sources:
  - query: |

        LET ValidateConfig(Config) = Config.Client.server_urls
          AND Config.Client.ca_certificate =~ "(?ms)-----BEGIN CERTIFICATE-----.+-----END CERTIFICATE-----"
          AND Config.Client.nonce

        LET CheckConfigPath(ConfigPath) = SELECT * FROM stat(filename=ConfigPath)
        LET Config &lt;=  parse_yaml(accessor="data", filename=ConfigYaml)

        LET DoIt = if(condition=ValidateConfig(Config=Config),
          else=log(level="ERROR", message="Config is invalid") AND FALSE,
          then=if(condition=CheckConfigPath(ConfigPath=ConfigPath).OSPath,
             else=log(level="ERROR",
                      message="Config Path %v is invalid",
                      args=ConfigPath) AND FALSE,
             then=copy(accessor="data", filename=ConfigYaml, dest=ConfigPath)
                AND if(condition= RekeyClient,
                then=log(message="Rekeying in %v seconds ", args=WaitPeriod)
                     AND rekey(wait=WaitPeriod),
                else=pskill(pid=getpid()))
        ))

        SELECT DoIt AS Success FROM scope()

    notebook:
    - name: Move a client to a different OrgId
      type: vql_suggestion
      template: |

        LET ClientId = "C.622d19ea21109231"
        LET RequiredOrgId = "O123"
        LET ConfigPath = "C:/Program Files/Velociraptor/client.config.yaml"

        SELECT _client_config AS Config, OrgId ,
            collect_client(artifacts="Admin.Client.UpdateClientConfig",
                           client_id=ClientId,
                           env=dict(ConfigYaml=_client_config,
                                    ConfigPath=ConfigPath))
        FROM orgs()
        WHERE OrgId = RequiredOrgId
        LIMIT 1

</code></pre>

