---
title: Windows.System.TaskScheduler
hidden: true
tags: [Client Artifact]
---

The Windows task scheduler is a common mechanism that malware uses
for persistence. It can be used to run arbitrary programs at a later
time. Commonly malware installs a scheduled task to run itself
periodically to achieve persistence.

This artifact enumerates all the task jobs (which are XML
files). The artifact uploads the original XML files and then
analyses them to provide an overview of the commands executed and
the user under which they will be run.


<pre><code class="language-yaml">
name: Windows.System.TaskScheduler
description: |
  The Windows task scheduler is a common mechanism that malware uses
  for persistence. It can be used to run arbitrary programs at a later
  time. Commonly malware installs a scheduled task to run itself
  periodically to achieve persistence.

  This artifact enumerates all the task jobs (which are XML
  files). The artifact uploads the original XML files and then
  analyses them to provide an overview of the commands executed and
  the user under which they will be run.

parameters:
  - name: TasksPath
    default: c:/Windows/System32/Tasks/**
  - name: AlsoUpload
    type: bool
    description: |
      If set we also upload the task XML files.
  - name: UploadCommands
    type: bool
    description: |
      If set we attempt to upload the commands that are
      mentioned in the scheduled tasks

sources:
  - name: Analysis
    query: |
      LET Uploads = SELECT Name, OSPath, if(
           condition=AlsoUpload='Y',
           then=upload(file=OSPath)) as Upload, Mtime
        FROM glob(globs=TasksPath)
        WHERE NOT IsDir

      // Job files contain invalid XML which confuses the parser - we
      // use regex to remove the invalid tags.
      LET parse_task = select OSPath, Mtime, parse_xml(
               accessor='data',
               file=regex_replace(
                    source=utf16(string=Data),
                    re='&lt;[?].+?&gt;',
                    replace='')) AS XML
        FROM read_file(filenames=OSPath)

      LET Results = SELECT OSPath, Mtime,
            XML.Task.Actions.Exec.Command as Command,
            expand(path=XML.Task.Actions.Exec.Command)  AS ExpandedCommand,
            XML.Task.Actions.Exec.Arguments as Arguments,
            XML.Task.Actions.ComHandler.ClassId as ComHandler,
            XML.Task.Principals.Principal.UserId as UserId,
            timestamp(epoch=XML.Task.Triggers.CalendarTrigger.StartBoundary) AS StartBoundary,
            XML as _XML
        FROM foreach(row=Uploads, query=parse_task)

      SELECT *,
         authenticode(filename=ExpandedCommand) AS Authenticode,
         if(condition=UploadCommands and ExpandedCommand,
            then=upload(file=ExpandedCommand)) AS Upload
      FROM Results

column_types:
- name: Upload
  type: upload_preview
- name: Authenticode
  type: collapsed

</code></pre>

