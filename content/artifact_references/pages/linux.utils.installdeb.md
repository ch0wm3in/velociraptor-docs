---
title: Linux.Utils.InstallDeb
hidden: true
tags: [Client Artifact]
---

Install a deb package and configure it with debconf answers. The package
may either be specified by name, as an uploaded file or as a "tool". If the
package already exists, it may be optionally reconfigured with debconf
answers.

There are three ways to specify a package (listed in order of preference if
all are set):

  - DebFile: An uploaded deb package.

  - DebTool: A deb package provided as a tool, specified by tool name. Since
    this is a utility artifact meant to be called by other artifacts, the
    tool should be specified in the artifact calling this artifact.
    Alternatively, configure the tool using
    [VQL](https://docs.velociraptor.app/vql_reference/server/inventory_add/).

  - DebName: The name of the package to install, or an absolute path to a deb
    file to install. Each word is considered as a package name or file name.
    `apt-get` interprets the package name, and allows you to specify a
    specific version, architecture, or even install and remove packages in
    the same go:

    - "foo": installs foo
    - "foo bar- baz=1.0.0-1 qux:arm64": installs foo, removes bar, installs
      a specific version of baz and a specific architecture of qux


<pre><code class="language-yaml">
name: Linux.Utils.InstallDeb
author: Andreas Misje – @misje
description: |
   Install a deb package and configure it with debconf answers. The package
   may either be specified by name, as an uploaded file or as a "tool". If the
   package already exists, it may be optionally reconfigured with debconf
   answers.

   There are three ways to specify a package (listed in order of preference if
   all are set):

     - DebFile: An uploaded deb package.

     - DebTool: A deb package provided as a tool, specified by tool name. Since
       this is a utility artifact meant to be called by other artifacts, the
       tool should be specified in the artifact calling this artifact.
       Alternatively, configure the tool using
       [VQL](https://docs.velociraptor.app/vql_reference/server/inventory_add/).

     - DebName: The name of the package to install, or an absolute path to a deb
       file to install. Each word is considered as a package name or file name.
       `apt-get` interprets the package name, and allows you to specify a
       specific version, architecture, or even install and remove packages in
       the same go:

       - "foo": installs foo
       - "foo bar- baz=1.0.0-1 qux:arm64": installs foo, removes bar, installs
         a specific version of baz and a specific architecture of qux

type: CLIENT

required_permissions:
   - EXECVE
   - FILESYSTEM_WRITE

reference:
   - https://manpages.debian.org/bookworm/debconf-doc/debconf-devel.7.en.html#Type

parameters:
   - name: DebName
     description: |
        Package to install (by name). Ignored if DebFile or DebTool is set. An
        absolute path to a deb file that already exists on the system is also
        accepted.

   - name: DebFile
     description: |
        Package to install (by file). Remember to click "Upload"! When set,
        DebName and DebTool is ignored. Use DebName with an absolute file path
        if the file already exists on the system and does not need to be
        uploaded.
     type: upload_file

   - name: DebTool
     description: |
        Package to install as a tool (tool name). The tool must be configured
        manually by using VQL or in another artifact (calling this artifact).
        Ignored if DebFile is set.

   - name: ToolSleepDuration
     description: |
        Maximum number of seconds to sleep before downloading the package. Only
        relevant if DebTool is set.
     type: int
     default: 20

   - name: UpdateSources
     description: |
        Run `apt-get update` before installing the package. This is not necessary
        if the package has no dependencies, and it should be disabled if there
        is no Internet.
     type: bool
     default: True

   - name: ForceConfNew
     type: bool
     description: |
        Use the configuration delivered by the package instead of keeping the
        local changes.

   - name: Reinstall
     type: bool
     description: |
        Reinstall the package if it is already installed. This is only useful
        if you know or suspect that the package installation is broken. This
        also reconfigures the package.

   - name: UpgradeOnly
     type: bool
     description: |
        Do not install the package; only upgrade it if it is already installed.

   - name: ReconfigureIfInstalled
     type: bool
     description: |
        If the package is already installed, run pre-seed debconf and
        `dpkg-reconfigure` instead.

   - name: DebConfValues
     type: csv
     description: |
        debconf is a system used by many packages for interactive configuration.
        When using a non-interactive frontend (like this artifact), answers may
        by provided as a "pre-seed" file. Example line:

        "wireshark-common/install-setuid,boolean,false"
     default: |
        Key,Type,Value

column_types:
  - name: Stdout
    type: nobreak

  - name: Stderr
    type: nobreak

sources:
  - precondition:
      SELECT OS From info() where OS = 'linux'

    query: |
       LET Tool = SELECT OSPath
         FROM Artifact.Generic.Utils.FetchBinary(ToolName=DebTool,
                                                 TemporaryOnly=true,
                                                 SleepDuration=ToolSleepDuration)
       LET Package &lt;= if(
           condition=DebTool,
           then=Tool[0].OSPath,
           else=if(
             condition=DebFile,
             /* apt requires file names to end in an architecture name, so
                create a copy ending in "_amd64.deb". The architecture chosen
                here, amd64, does not need to match the architecture of neither
                the package or the system:
             */
             then=copy(dest=tempdir() + '/package_amd64.deb',
                       filename=DebFile),
                       else=DebName))

       /* The file name is lost from the uploaded file, so extract it from the
          package instead (apt has certain requirements for the file name):
       */
       LET PackageInfo = SELECT Stdout
         FROM execve(argv=['/usr/bin/dpkg-deb', '--field', Package, 'Package'])

       LET PackageName = if(condition=DebTool OR DebFile,
                            // remove "\n":
                            then=PackageInfo[0].Stdout[:-1], else=DebName)

       /* The file format is "package_name question type answer": */
       LET PreSeedLines = SELECT join(sep=' ',
           array=(PackageName, Key, Type, Value)) AS Line
         FROM DebConfValues

       LET PreSeedFile &lt;= tempfile(data=join(sep='\n', array=PreSeedLines.Line))

       LET AptEnv = dict(
           DEBIAN_FRONTEND='noninteractive',
           DEBCONF_NOWARNINGS='yes')

       LET AptOpts &lt;= ('-f', '-y', '-o', 'Debug::pkgProblemResolver=yes',
                       '--no-install-recommends') +
           if(condition=ForceConfNew,
              then=('-o', 'Dpkg::Options::=--force-confnew'), else=[]) +
           if(condition=Reinstall, then=('--reinstall', ), else=[]) +
           if(condition=UpgradeOnly, then=('--only-upgrade', ), else=[])

       LET PreSeed = SELECT 'Pre-seed debconf' AS Step, *
         FROM if(condition=DebConfValues, then={
             SELECT *
             FROM execve(argv=['/usr/bin/debconf-set-selections', PreSeedFile, ])
             WHERE log(message='Pre-seeding %v', dedup= -1,
                       args=PackageName,
                       level='INFO')
              AND (NOT ReturnCode OR log(level='ERROR',
                      message='%v failed: %v', args=(Step, Stderr)))
           })

       /* Install regardless of whether package is installed or not, handing all
          the (arch-specific) version comparison logic to apt:
        */
       LET Install &lt;= SELECT * FROM chain(
           a_update={
             SELECT 'Updating index' AS Step, *
             FROM if(condition=UpdateSources, then={
                 SELECT *
                 FROM execve(argv=['/usr/bin/apt-get', '-y', 'update'])
                 WHERE log(message='Updating package index before installing',
                           level='INFO')
               })
           },
           b_debconf=PreSeed,
           c_install={
             SELECT 'Installing package' AS Step, *
             FROM execve(
               argv=('/usr/bin/apt-get', ) + AptOpts +
               if(condition=Package = DebName,
                  then=('install', ) + split(sep='''\s+''', string=Package),
                  else=('install', Package)),
               env=AptEnv)
             WHERE log(message='Installing deb package %v',
                       args=PackageName,
                       dedup= -1,
                       level='INFO')
           })
         WHERE NOT ReturnCode OR log(level='ERROR',
                                     message='%v failed: %v',
                                     args=(Step, Stderr))

       SELECT * FROM chain(
         a_install=Install,
         b_reconfigure={
           SELECT * FROM if(
             condition=ReconfigureIfInstalled AND (
                Reinstall OR Install.Stdout =~ 'Skipping|already the newest version'),
             then={
               SELECT 'Reconfiguring package' AS Step, *
               FROM execve(argv=['/usr/sbin/dpkg-reconfigure', PackageName, ],
                           env=AptEnv)
               WHERE log(message='Reconfiguring deb package %v',
                         args=PackageName,
                         dedup= -1,
                         level='INFO')
                AND (NOT ReturnCode OR log(level='ERROR',
                                       message='%v failed: %v',
                                       args=(Step, Stderr)))
             })
         })

</code></pre>

