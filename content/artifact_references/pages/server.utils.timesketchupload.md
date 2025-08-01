---
title: Server.Utils.TimesketchUpload
hidden: true
tags: [Server Artifact]
---

Timesketch is an interactive collaborative timeline analysis tool
that can be found at https://timesketch.org/

This artifact uploads Velociraptor's timelines to Timesketch using
the Timesketch client library. The artifact assumes the client
library is installed and configured on the server.

To install the Timesketch client library:
```
pip install timesketch-import-client timesketch-cli-client
```

To configure the client library to access your Timesketch instance
see instructions https://timesketch.org/guides/user/cli-client/ and
https://timesketch.org/guides/user/upload-data/

This artifact assumes that the timesketch CLI is preconfigured with
the correct credentials in the `.timesketchrc` file.


<pre><code class="language-yaml">
name: Server.Utils.TimesketchUpload
description: |
  Timesketch is an interactive collaborative timeline analysis tool
  that can be found at https://timesketch.org/

  This artifact uploads Velociraptor's timelines to Timesketch using
  the Timesketch client library. The artifact assumes the client
  library is installed and configured on the server.

  To install the Timesketch client library:
  ```
  pip install timesketch-import-client timesketch-cli-client
  ```

  To configure the client library to access your Timesketch instance
  see instructions https://timesketch.org/guides/user/cli-client/ and
  https://timesketch.org/guides/user/upload-data/

  This artifact assumes that the timesketch CLI is preconfigured with
  the correct credentials in the `.timesketchrc` file.

required_permissions:
  - EXECVE

parameters:
  - name: NotebookId
    description: The notebook ID that contains the super timeline
  - name: SuperTimelineName
    description: The name of the super timeline
  - name: TimelineName
    description: The name of the timeline within the super timeline.
  - name: SketchName
    description: The name of the sketch to upload to
  - name: TimesketchCLICommand
    default: "timesketch"
    description: |
      The path to the timesketch cli binary. If you installed in a
      virtual environment this will be inside that environment.

type: SERVER

export: |
  LET timesketch_import_command = TimesketchCLICommand + "_importer"

  -- The uploader tool can create a new "Sketch" but if we want to
  -- just add a timeline to an existing sketch we need to specify the
  -- ID. This function finds the ID for the specified Sketch if it
  -- exists. NOTE that you can have multiple Sketches with the same
  -- name! We pick the first.
  LET GetIdToSketch(Sketch) = SELECT * FROM foreach(row={
    SELECT Stdout
      FROM execve(
          argv=[TimesketchCLICommand, "--output-format",
              "json", "sketch", "list"], length=10000)
  }, query={
    SELECT * FROM parse_json_array(data=Stdout)
  })
  WHERE name = Sketch

  -- Enumerate all the timelines in a super timeline
  LET _GetAllTimelines(SuperTimelineName, NotebookId) = SELECT *
   FROM foreach(row={
     SELECT *
     FROM timelines(notebook_id=NotebookId)
     WHERE name = SuperTimelineName
  }, query={ SELECT * FROM timelines })

  LET _GetTimelineMetdata(SuperTimelineName, NotebookId, TimelineName) =
  SELECT * FROM _GetAllTimelines(
      SuperTimelineName=SuperTimelineName, NotebookId=NotebookId)
  WHERE Id = TimelineName

  -- Gets the metadata of a named timeline
  LET GetTimelineMetdata(SuperTimelineName, NotebookId, TimelineName) =
     _GetTimelineMetdata(SuperTimelineName= SuperTimelineName,
                         NotebookId = NotebookId,
                         TimelineName=TimelineName)[0]

  -- Timesketch insists the file have the .csv extension.
  LET tmp &lt;= tempfile(extension=".csv")

  -- We copy the timeline to a temp csv file then upload that. This
  -- might seem inefficient but timesketch is written in python so it
  -- is already very slow. The extra tempfile does not make much
  -- difference in practice.
  LET WriteTmpFile(NotebookId, SuperTimelineName, TimelineName) =
       SELECT count() AS Count
       FROM write_csv(filename=tmp, query={
          SELECT Timestamp as timestamp, Message as message, *
          FROM timeline(notebook_id=NotebookId, timeline=SuperTimelineName,
                        components=TimelineName)
       })
       GROUP BY 1

  LET ImportToTS(SuperTimelineName, NotebookId, TimelineName, SketchName) =
  SELECT * FROM chain(a={
     SELECT format(format="Exporting %v rows to %v", args=[WriteTmpFile(
       NotebookId=NotebookId, SuperTimelineName=SuperTimelineName,
       TimelineName=TimelineName)[0].Count, tmp]) AS Stdout
     FROM scope()
  }, c={
    SELECT * FROM foreach(row={

      -- This is unfortunately slow and unnecessary but Timesketch
      -- does not have a flag that just says - add timeline to
      -- existing sketch. So we have to type to find the sketch ID
      -- first.
      SELECT GetIdToSketch(Sketch=SketchName)[0].id || 0 AS SketchID
      FROM scope()

    }, query={

      -- Launch the import library and display the output.
      SELECT Stdout, Stderr, SketchID,
             SketchName, TimelineName
      FROM execve(argv=[timesketch_import_command, "--sketch_name",
                        SketchName, "--sketch_id", SketchID,
                        "--timeline_name", TimelineName,
                        tmp], sep="\n")
    })
  })

sources:
  - query: |
      SELECT * FROM ImportToTS(
         SuperTimelineName=SuperTimelineName,
         NotebookId=NotebookId,
         TimelineName=TimelineName,
         SketchName=SketchName)

</code></pre>

