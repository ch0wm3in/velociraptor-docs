---
title: notebook_export
index: true
noTitle: true
no_edit: true
---



<div class="vql_item"></div>


## notebook_export
<span class='vql_type label label-warning pull-right page-header'>Function</span>



<div class="vqlargs"></div>

Arg | Description | Type
----|-------------|-----
notebook_id|The id of the notebook to export|string (required)
filename|The name of the export. If not set this will be named according to the notebook id and timestamp|string
type|Set the type of the export (html or zip).|string

<span class="permission_list vql_type">Required permissions:</span><span class="permission_list linkcolour label label-important">PREPARE_RESULTS</span>

### Description

Exports a notebook to a zip file or HTML.
