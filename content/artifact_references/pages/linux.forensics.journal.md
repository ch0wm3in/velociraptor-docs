---
title: Linux.Forensics.Journal
hidden: true
tags: [Client Artifact]
---

Parses the binary journal logs. Systemd uses a binary log format to
store logs.


<pre><code class="language-yaml">
name: Linux.Forensics.Journal
description: |
  Parses the binary journal logs. Systemd uses a binary log format to
  store logs.

parameters:
- name: JournalGlob
  type: glob
  description: A Glob expression for finding journal files.
  default: /{run,var}/log/journal/*/*.journal

- name: AlsoUpload
  type: bool
  description: If set we also upload the raw files.

sources:
- name: Uploads
  query: |
    SELECT * FROM if(condition=AlsoUpload,
    then={
       SELECT OSPath, upload(file=OSPath) AS Upload
       FROM glob(globs=JournalGlob)
    })

- query: |
    SELECT * FROM foreach(row={
      SELECT OSPath FROM glob(globs=JournalGlob)
    }, query={
      SELECT *
      FROM parse_journald(filename=OSPath)
    })

</code></pre>

