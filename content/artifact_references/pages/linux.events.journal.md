---
title: Linux.Events.Journal
hidden: true
tags: [Client Event Artifact]
---

Watches the binary journal logs. Systemd uses a binary log format to
store logs.


<pre><code class="language-yaml">
name: Linux.Events.Journal
description: |
  Watches the binary journal logs. Systemd uses a binary log format to
  store logs.

type: CLIENT_EVENT

parameters:
- name: JournalGlob
  type: glob
  description: A Glob expression for finding journal files.
  default: /{run,var}/log/journal/*/*.journal

sources:
- query: |
    SELECT * FROM foreach(row={
      SELECT OSPath FROM glob(globs=JournalGlob)
    }, query={
      SELECT *
      FROM watch_journald(filename=OSPath)
    }, workers=100)

</code></pre>

