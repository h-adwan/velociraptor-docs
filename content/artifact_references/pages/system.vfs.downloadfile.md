---
title: System.VFS.DownloadFile
hidden: true
tags: [Client Artifact]
---

This is an internal artifact used by the GUI to populate the
VFS. You may run it manually if you like, but typically it is
launched by the GUI when the user clicks the "Collect from client"
button at the file "Stats" tab.

If you run it yourself (or via the API) the results will also be
shown in the VFS view.


```yaml
name: System.VFS.DownloadFile
description: |
  This is an internal artifact used by the GUI to populate the
  VFS. You may run it manually if you like, but typically it is
  launched by the GUI when the user clicks the "Collect from client"
  button at the file "Stats" tab.

  If you run it yourself (or via the API) the results will also be
  shown in the VFS view.

parameters:
  - name: Path
    description: The path of the file to download.
    default: /
  - name: Components
    type: json_array
    description: Alternatively, this is an explicit list of components.
  - name: Accessor
    default: file
  - name: Recursively
    type: bool
    description: |
      If specified, Path is interpreted as a directory and
      we download all files below it.

sources:
  - query: |
      LET download_one_file = if(
         condition=version(plugin="stat") > 1,
         then= {
           SELECT OSPath AS Path, Accessor,
              Size, upload(file=OSPath, accessor=Accessor) AS Upload
           FROM stat(filename=Components, accessor=Accessor)
        },
        else= {
           SELECT FullPath AS Path, Accessor,
              Size, upload(file=FullPath, accessor=Accessor) AS Upload
          FROM stat(filename=Path, accessor=Accessor)
        })

      LET download_recursive = if(
         condition=version(plugin="stat") > 1,
         then= {
           SELECT OSPath AS Path, Accessor,
              Size, upload(file=OSPath, accessor=Accessor) AS Upload
           FROM glob(globs="**", root=Components,
                     accessor=Accessor, nosymlink=TRUE)
           WHERE Mode.IsRegular
        },
        else={
          SELECT FullPath AS Path, Accessor,
            Size, upload(file=FullPath, accessor=Accessor) AS Upload
          FROM glob(globs="**", root=Path, accessor=Accessor)
          WHERE Mode.IsRegular
       })

      SELECT Path, Accessor,
             Upload.Size AS Size,
             Upload.StoredSize AS StoredSize,
             Upload.Sha256 AS Sha256,
             Upload.Md5 AS Md5,
             Path.Components AS _Components
      FROM if(condition=Recursively,
        then={ SELECT * FROM download_recursive},
        else={ SELECT * FROM download_one_file})

```
