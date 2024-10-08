---
title: Server.Internal.TimelineAdd
hidden: true
tags: [Server Event Artifact]
---

This artifact will fire whenever a timeline is added to a super
timeline. You can use this to monitor for users adding timelines and
forward them to an external timeline system (e.g. TimeSketch)


<pre><code class="language-yaml">
name: Server.Internal.TimelineAdd
type: SERVER_EVENT
description: |
  This artifact will fire whenever a timeline is added to a super
  timeline. You can use this to monitor for users adding timelines and
  forward them to an external timeline system (e.g. TimeSketch)

column_types:
  - name: NotebookId
  - name: SuperTimelineName
  - name: Timeline

  # What type of event this is: can be Delete, AddTimeline
  - name: Action

</code></pre>

