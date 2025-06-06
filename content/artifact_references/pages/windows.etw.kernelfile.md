---
title: Windows.ETW.KernelFile
hidden: true
tags: [Client Event Artifact]
---

This artifact follows the Microsoft-Windows-Kernel-File provider.

NOTE: We can only attach to this provider when running as
NT_USER/SYSTEM.


<pre><code class="language-yaml">
name: Windows.ETW.KernelFile
description: |
  This artifact follows the Microsoft-Windows-Kernel-File provider.

  NOTE: We can only attach to this provider when running as
  NT_USER/SYSTEM.

aliases:
  - Windows.ETW.FileCreation

type: CLIENT_EVENT

references:
  - https://github.com/repnz/etw-providers-docs/blob/master/Manifests-Win10-18990/Microsoft-Windows-Kernel-File.xml

parameters:
  - name: ProcessRegex
    type: regex
    description: View Processes with Executables matching this regex
    default: .

  - name: IgnoreProcessRegex
    type: regex
    description: Ignore Processes with Executables matching this regex

  - name: Events
    type: multichoice
    description: Events to view
    default: '["NameCreate", "NameDelete", "FileOpen", "Rename", "RenamePath", "CreateNewFile"]'
    choices:
      - NameCreate
      - NameDelete
      - FileOpen
      - Rename
      - RenamePath
      - CreateNewFile

sources:
  - query: |
      -- KERNEL_FILE_KEYWORD_FILENAME | KERNEL_FILE_KEYWORD_CREATE | KERNEL_FILE_KEYWORD_DELETE_PATH
      LET Keyword &lt;= 0x1490
      LET EIDLookup &lt;= dict(
        `10`="NameCreate", `11`="NameDelete", `12`="FileOpen",
        `19`="Rename", `27`="RenamePath",`30`="CreateNewFile")

      LET ETW = SELECT *
      FROM watch_etw(guid='{edd08927-9cc4-4e65-b970-c2560fb5c289}',
           description="Microsoft-Windows-Kernel-File", any=Keyword)

      SELECT System.ID AS EID,
         get(item=EIDLookup, field=str(str=System.ID)) AS EventType,
         process_tracker_get(id=System.ProcessID).Data AS ProcInfo,
         process_tracker_callchain(id=System.ProcessID).Data.Exe AS CallChain,
         EventData
      FROM delay(query=ETW, delay=3)
      WHERE EventType IN Events
        AND ProcInfo.Exe =~ ProcessRegex
        AND if(condition=IgnoreProcessRegex,
               then=NOT ProcInfo.Exe =~ IgnoreProcessRegex,
               else=TRUE)

</code></pre>

