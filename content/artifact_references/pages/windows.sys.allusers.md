---
title: Windows.Sys.AllUsers
hidden: true
tags: [Client Artifact]
---

List User accounts. We combine two data sources - the output from
the `NetUserEnum` API (termed `local` users) and the list of SIDs in
the registry (termed `remote` users).

In this artifact, 'remote' means that user profile was cached in the
registry, but the user does not appear in the output of the
`NetUserEnum` API - this normally happens for users remotely logging
into the system using domain credentials.

On Domain Controllers the `NetUserEnum` API will return the contents
of the entire ActiveDirectory as a list of 'local' users, however
this does not mean that the users have logged into the DC
locally. In this artifact we limit the number of users to 1000. If
you need to obtain the full list from the AD, customize this
artifact.


<pre><code class="language-yaml">
name: Windows.Sys.AllUsers
description: |
  List User accounts. We combine two data sources - the output from
  the `NetUserEnum` API (termed `local` users) and the list of SIDs in
  the registry (termed `remote` users).

  In this artifact, 'remote' means that user profile was cached in the
  registry, but the user does not appear in the output of the
  `NetUserEnum` API - this normally happens for users remotely logging
  into the system using domain credentials.

  On Domain Controllers the `NetUserEnum` API will return the contents
  of the entire ActiveDirectory as a list of 'local' users, however
  this does not mean that the users have logged into the DC
  locally. In this artifact we limit the number of users to 1000. If
  you need to obtain the full list from the AD, customize this
  artifact.

parameters:
  - name: remoteRegKey
    default: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\*

export: |
  -- Cache function for lookupSID
  LET LookupSIDCache(SID) = cache(name="SID", key=SID,
     func=lookupSID(sid=SID) ||

     -- resolve usernames via registry if lookupSID is not available
     -- or yields no results

          pathspec(parse=stat(accessor="registry",
               filename="HKEY_LOCAL_MACHINE/Software/Microsoft/Windows NT/CurrentVersion/ProfileList/" +
                      SID + "/ProfileImagePath").Data.value).Basename || "")

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    query: |
        LET GetTimestamp(High, Low) = if(condition=High,
                then=timestamp(winfiletime=High * 4294967296 + Low))

        -- lookupSID() may not be available on deaddisk analysis
        LET roaming_users &lt;=
          SELECT
             split(string=Key.OSPath.Basename, sep="-")[-1] as Uid,
             "" AS Gid,
             LookupSIDCache(SID=Key.OSPath.Basename || "") AS Name,
             Key.OSPath as Description,
             ProfileImagePath as Directory,
             Key.OSPath.Basename as UUID,
             Key.Mtime as Mtime,
             {
                SELECT Mtime
                FROM stat(filename=expand(path=ProfileImagePath))
             } AS HomedirMtime,
             dict(ProfileLoadTime=GetTimestamp(
                  High=LocalProfileLoadTimeHigh, Low=LocalProfileLoadTimeLow),
                  ProfileUnloadTime=GetTimestamp(
                  High=LocalProfileUnloadTimeHigh, Low=LocalProfileUnloadTimeLow),
                  ProfileImagePath=ProfileImagePath,
                  ProfileKey=Key.OSPath
             ) AS Data
           FROM read_reg_key(globs=remoteRegKey, accessor="registry")


        LET roaming_users_lookup &lt;= memoize(query=roaming_users, key="UUID")

        -- On a DC the NetUserEnum API will return the entire domain!
        LET local_users &lt;= select User_id as Uid,
           Primary_group_id as Gid, Name,
           Comment as Description,
           get(item=roaming_users_lookup, field=User_sid) AS  RoamingData,
           User_sid as UUID
        FROM users()
        LIMIT 1000

        LET local_users_lookup &lt;= memoize(query={
            SELECT UUID FROM local_users
        }, key="UUID")

        -- Populate the mtime from the user's home directory.
        LET local_users_with_mtime = SELECT Uid, Gid, Name, Description,
            RoamingData.Directory AS Directory,
            UUID,
            RoamingData.Mtime As Mtime,
            RoamingData.HomedirMtime AS HomedirMtime,
            RoamingData.Data || dict() AS Data
        FROM local_users

        SELECT * from chain(
           q1=local_users_with_mtime,
           q2={
             SELECT * FROM roaming_users
             -- Only show records who were not shown before
             WHERE NOT get(item=local_users_lookup, field=UUID)
           })
         --ORDER BY Gid DESC
    notebook:
      - type: VQL
        template: |
          /*
          # Users Hunt

          Enumerating all the users on all endpoints can reveal machines
          which had an unexpected login activity. For example, if a user
          from an unrelated department is logging into an endpoint by
          virtue of domain credentials, this could mean their account is
          compromised and the attackers are laterally moving through the
          network.
          */
          LET s = scope()
          SELECT Name, UUID, s.Fqdn AS Fqdn, HomedirMtime as LastMod, Data FROM source()
          WHERE NOT UUID =~ "(-5..$|S-1-5-18|S-1-5-19|S-1-5-20)"

</code></pre>

