---
title: Generic.Client.DiskSpace
hidden: true
tags: [Client Artifact]
---

This artifact reports the amount of free disk space. It is designed
to work equally on all architectures:

  1. On Linux and MacOS we call `df -h`.
  2. On Windows we use WMI


<pre><code class="language-yaml">
name: Generic.Client.DiskSpace
description: |
  This artifact reports the amount of free disk space. It is designed
  to work equally on all architectures:

    1. On Linux and MacOS we call `df -h`.
    2. On Windows we use WMI

implied_permissions:
  - EXECVE

sources:
- query: |
    LET NonWindows = SELECT * FROM foreach(row={
      SELECT regex_replace(source=Stdout, re="( on| +)", replace=" ") AS Stdout
      FROM execve(argv=["df", "-h"], length=10000)
    }, query={
      SELECT * FROM parse_csv(accessor="data", filename=Stdout, separator=" ")
    })

    -- WMI returns these as strings, we need to convert to ints
    LET wmi_query = SELECT *,
         int(int=FreeSpace) AS FreeSpace,
         int(int=Size) AS Size
      FROM wmi(query="SELECT * FROM Win32_LogicalDisk")

    LET Windows = SELECT DeviceID, Description,
           VolumeName, VolumeSerialNumber,
           humanize(bytes=Size) AS Size,
           humanize(bytes=FreeSpace) AS FreeSpace,
           int(int=FreeSpace / Size * 100) AS `Free%`
    FROM wmi_query

    SELECT * FROM if(condition={
      SELECT OS FROM info() WHERE OS =~ "windows"
    },
    then={ SELECT * FROM Windows},
    else={ SELECT * FROM NonWindows})

</code></pre>

