---
title: Linux.Sys.BashShell
hidden: true
tags: [Client Artifact]
---

This artifact allows running arbitrary commands through the system
shell.

Since Velociraptor typically runs as root, the commands will also
run as root.

This is a very powerful artifact since it allows for arbitrary
command execution on the endpoints. Therefore this artifact requires
elevated permissions (specifically the `EXECVE`
permission). Typically it is only available with the `administrator`
role.


<pre><code class="language-yaml">
name: Linux.Sys.BashShell
description: |
  This artifact allows running arbitrary commands through the system
  shell.

  Since Velociraptor typically runs as root, the commands will also
  run as root.

  This is a very powerful artifact since it allows for arbitrary
  command execution on the endpoints. Therefore this artifact requires
  elevated permissions (specifically the `EXECVE`
  permission). Typically it is only available with the `administrator`
  role.

required_permissions:
  - EXECVE

parameters:
  - name: Command
    default: "ls -l /"

sources:
  - query: |
      LET SizeLimit &lt;= 4096
      SELECT if(condition=len(list=Stdout) &lt; SizeLimit,
                then=Stdout) AS Stdout,
             if(condition=len(list=Stdout) &gt;= SizeLimit,
                then=upload(accessor="data",
                            file=Stdout, name="Stdout")) AS StdoutUpload
      FROM execve(argv=["/bin/bash", "-c", Command],
                  length=10000000)

column_types:
- name: StdoutUpload
  type: preview_upload

</code></pre>

