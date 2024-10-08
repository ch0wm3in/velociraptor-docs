---
title: Windows.Carving.USN
hidden: true
tags: [Client Artifact]
---

Carve URN Journal records from the disk.

The USN journal is a very important source of information about when
and how files were manipulated on the filesystem. However, typically
the journal is rotated within a few days.

This artifact carves out USN journal entries from the raw disk. This
might recover older entries which have since been rotated from the
journal file.

## Notes

1. Like all carving, USN carving is not very reliable. You
   would tend to use it to corroborate an existing theory or to
   discover new leads.

2. This artifact takes a long time to complete - you should
   probably increase the collection timeout past 10 minutes (usually
   more than an hour).

3. The reassembled OSPath is derived from the MFTId referenced in
   the USN record. Bear in mind that this might be out of date and
   inaccurate.

4. If you need to carve from a standalone file (e.g. collection from
   `Windows.KapeFiles.Targets`) you should use the
   Windows.Carving.USNFiles artifact instead.


<pre><code class="language-yaml">
name: Windows.Carving.USN
description: |
  Carve URN Journal records from the disk.

  The USN journal is a very important source of information about when
  and how files were manipulated on the filesystem. However, typically
  the journal is rotated within a few days.

  This artifact carves out USN journal entries from the raw disk. This
  might recover older entries which have since been rotated from the
  journal file.

  ## Notes

  1. Like all carving, USN carving is not very reliable. You
     would tend to use it to corroborate an existing theory or to
     discover new leads.

  2. This artifact takes a long time to complete - you should
     probably increase the collection timeout past 10 minutes (usually
     more than an hour).

  3. The reassembled OSPath is derived from the MFTId referenced in
     the USN record. Bear in mind that this might be out of date and
     inaccurate.

  4. If you need to carve from a standalone file (e.g. collection from
     `Windows.KapeFiles.Targets`) you should use the
     Windows.Carving.USNFiles artifact instead.

parameters:
  - name: Device
    default: "C:"
    description: The NTFS drive to carve
  - name: MFTFile
    description: Alternatively provide an MFTFile to use for resolving paths.
  - name: USNFile
    description: Alternatively provide a previously extracted USN file to carve or an image file.
  - name: Accessor
    description: The accessor to use.
  - name: FileNameRegex
    description: "Regex search over File Name"
    default: "."
    type: regex
  - name: DateAfter
    type: timestamp
    description: "search for events after this date. YYYY-MM-DDTmm:hh:ssZ"
  - name: DateBefore
    type: timestamp
    description: "search for events before this date. YYYY-MM-DDTmm:hh:ssZ"

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
        -- firstly set timebounds for performance
        LET DateAfterTime &lt;= if(condition=DateAfter,
             then=DateAfter, else="1600-01-01")
        LET DateBeforeTime &lt;= if(condition=DateBefore,
            then=DateBefore, else="2200-01-01")

        -- If the user specified an MFTFile then ignore the device
        LET Device &lt;= if(condition=MFTFile OR USNFile, then=NULL,
          else=if(condition=Device,
          then=pathspec(parse=Device, path_type="ntfs")))

        LET Parse(MFT, USN, Accessor) = SELECT *
              FROM carve_usn(accessor=Accessor,
                             mft_filename=MFT, usn_filename=USN)
              WHERE Filename =~ FileNameRegex
                AND Timestamp &lt; DateBeforeTime
                AND Timestamp &gt; DateAfterTime

        SELECT *
        FROM if(condition=Device, then={
          SELECT Timestamp,
            Filename,
            Device + OSPath AS OSPath,
            _Links,
            Reason,
            _FileMFTID as MFTId,
            _FileMFTSequence as Sequence,
            _ParentMFTID as ParentMFTId,
            _ParentMFTSequence as ParentSequence,
            FileAttributes,
            SourceInfo,
            Usn
          FROM Parse(Accessor="ntfs",
              MFT=Device + "$MFT",
              USN=Device)
        }, else={
          SELECT Timestamp,
            Filename,
            OSPath,
            _Links,
            Reason,
            _FileMFTID as MFTId,
            _FileMFTSequence as Sequence,
            _ParentMFTID as ParentMFTId,
            _ParentMFTSequence as ParentSequence,
            FileAttributes,
            SourceInfo,
            Usn
          FROM Parse(Accessor=Accessor,
              MFT=MFTFile, USN=USNFile)
        })

</code></pre>

