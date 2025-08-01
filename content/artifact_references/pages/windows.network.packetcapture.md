---
title: Windows.Network.PacketCapture
hidden: true
tags: [Client Artifact]
---

Run this artifact twice, the first time, set the StartTrace flag to
True to start the PCAP collection, this will have the VQL return a
single row (the TraceFile generated) When you want to stop
collecting, and transform this TraceFile to a PCAP, re-run this
artifact with StartTrace as false, and put path of the .etl file
created in the previous step in the TraceFile. This will then
convert the .etl to a PCAP and upload it.


<pre><code class="language-yaml">
name: Windows.Network.PacketCapture
author: Cybereason &lt;omer.yampel@cybereason.com&gt;
description: |
  Run this artifact twice, the first time, set the StartTrace flag to
  True to start the PCAP collection, this will have the VQL return a
  single row (the TraceFile generated) When you want to stop
  collecting, and transform this TraceFile to a PCAP, re-run this
  artifact with StartTrace as false, and put path of the .etl file
  created in the previous step in the TraceFile. This will then
  convert the .etl to a PCAP and upload it.

precondition: SELECT OS From info() where OS = 'windows'

tools:
    - name: etl2pcapng
      url: https://github.com/microsoft/etl2pcapng/releases/download/v1.4.0/etl2pcapng.zip

implied_permissions:
  - FILESYSTEM_WRITE
  - EXECVE

parameters:
    - name: StartTrace
      type: bool
      default: Y
    - name: TraceFile
      type: string
      default:

sources:
    - query: |
        LET tool_zip = SELECT * FROM Artifact.Generic.Utils.FetchBinary(
            ToolName="etl2pcapng", IsExecutable=FALSE)

        LET ExePath &lt;= tempfile(extension='.exe')

        LET etl2pcapbin &lt;= SELECT
            copy(
              filename=pathspec(
                 DelegatePath=tool_zip[0].OSPath,
                 Path="etl2pcapng/x64/etl2pcapng.exe"),
              dest=ExePath,
              accessor='zip'
            ) AS file
        FROM scope()

        LET outfile &lt;= tempfile(extension=".pcapng")

        LET stop_trace = SELECT * FROM execve(
             argv=['netsh', 'trace', 'stop'])

        LET convert_pcap = SELECT * FROM execve(
             argv=[etl2pcapbin[0].file, TraceFile, outfile])

        LET end_trace = SELECT * FROM chain(
                a=stop_trace,
                b=convert_pcap,
                c={SELECT upload(file=outfile) AS Upload FROM scope()},
                d={SELECT upload(file=TraceFile) AS Upload FROM scope()}
            )

        LET launch_trace =
                SELECT
                    split(string=split(
                        string=Stdout,
                        sep="Trace File: ")[1],
                    sep="\r\nAppend:")[0] as etl_file
                FROM execve(argv=["netsh", "trace", "start", "capture=yes"])
                WHERE log(message="stderr: " + Stderr), log(message="stdout: " + Stdout)

        SELECT * FROM if(
                condition=StartTrace,
                then={ SELECT * FROM launch_trace},
                else={ SELECT * FROM end_trace }
        )

</code></pre>

