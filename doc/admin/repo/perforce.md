# Using Perforce depots with Sourcegraph

Sourcegraph supports [Perforce Helix](https://www.perforce.com/solutions/version-control) depots using [p4-fusion](https://github.com/salesforce/p4-fusion). This creates an equivalent Git repository from a Perforce depot, which can then be indexed by Sourcegraph.

![Screenshot of a Perforce repository in a Sourcegraph](https://sourcegraphstatic.com/git-p4-example.png)

## Configure experimental features

As of Sourcegraph 5.1, there are new features for Perforce depots that need to be enabled.

### Changelist Id in URLs

Add `"perforceChangelistMapping": "enabled",` to `experimentalFeatures` in the [site configuration](../config/site_config.md):

```json
{
  "experimentalFeatures": {
   "perforceChangelistMapping": "enabled",
  }
}
```

When it is enabled, Sourcegraph URLs for Perforce code hosts will use the Changelist Id instead of commit hashes.

### Batch Changes support for Perforce depots

Add `"batchChanges.enablePerforce": true` to `experimentalFeatures` in the [site configuration](../config/site_config.md):

```json
{
  "experimentalFeatures": {
   "batchChanges.enablePerforce": true,
  }
}
```

Batch Changes does not support repos that use sub-repo permissions, so in order to use batch changes with Perforce depots, the code host cannot use [file-level permissions](#file-level-permissions).

When a Batch Change is published, it is sent as a shelved changelist to the server configured in the code host. The Changelist Id is displayed in the UI for the user to use for managing the shelved changelist.

## Add a Perforce code host connection

Perforce depots can be added to a Sourcegraph instance by adding the appropriate [code host connection](../external_service/index.md).

To enable Perforce code host connections, a site admin must:

1. Go to **Site admin > Manage code hosts > Add code host**

1. Scroll down the list of supported code hosts and select **Perforce**.

1. Configure which depots are mirrored/synchronized as Git repositories to Sourcegraph:

    - [`depots`](perforce.md#depots)
      
      A list of depot paths that can be either a depot root or an arbitrary subdirectory. **Note**: Only `"local"` type depots are supported.

    - [`p4.user`](perforce.md#p4-user)
      
      The user to be authenticated for `p4` CLI, and should be capable of performing:
      - `p4 login`
      - `p4 trust`
      - and any p4 commands involved with `git p4 clone` and `git p4 sync` for listed `depots`.
           
      If repository permissions are mirrored, the user needs additional ability (aka. "super" access level) to perform the commands:
      - `p4 protects`
      - `p4 groups`
      - `p4 group`
      - `p4 users`
   
    - [`p4.passwd`](perforce.md#p4-passwd)
      
      The ticket to be used for authenticating the `p4.user`. It is recommended to create tickets of users in a group that never expire. Use the command `p4 -u <p4.user> login -p -a` to obtain a ticket value.
      
    - See the [configuration documentation below](#configuration) for other fields you can configure.

1. Configure `fusionClient`:

    ```json
    {
      "fusionClient": {
        "enabled": true,
        "lookAhead": 2000
      }
    }
    ```

    > NOTE: While the `fusionClient` configuration is optional, without it the code host connection uses `git p4`, which has performance issues so we strongly recommend `p4-fusion`.

    Details of all `p4-fusion` configuration fields can be seen [here](https://sourcegraph.com/github.com/sourcegraph/sourcegraph@2a716bd70c294acf1b3679b790834c4dea9ea956/-/blob/schema/perforce.schema.json?L84-147).

1. Click **Add repositories**.

Sourcegraph will now talk to the Perforce host and sync the configured `depots` to the Sourcegraph instance.

It's worthwhile to note some limitations of this process:

- When syncing depots either [git p4](https://git-scm.com/docs/git-p4) or [p4-fusion](https://github.com/salesforce/p4-fusion) (recommended) are used to convert Perforce depots into git repositories so that Sourcegraph can index them.
- Rename of a Perforce depot, including changing the depot on the Perforce server or the `repositoryPathPattern` config option, will cause a re-import of the depot.
- Unless [permissions syncing](#repository-permissions) is enabled, Sourcegraph is not aware of the depot permissions, so it can't enforce access restrictions.

## Repository permissions

To enforce file-level permissions for Perforce depots using the [Perforce protects file](https://www.perforce.com/manuals/cmdref/Content/CmdRef/p4_protect.html), include [the `authorization` field](https://sourcegraph.com/github.com/sourcegraph/sourcegraph@2a716bd70c294acf1b3679b790834c4dea9ea956/-/blob/schema/perforce.schema.json?L67-78) in the configuration of the Perforce code host connection you created [above](#add-a-perforce-code-host):

```json
{
  "authorization": {}
}
```

Adding the `authorization` field to the code host connection configuration will enable partial parsing of the protects file. [Learn more about the partial support of protects file parsing](#known-issues-and-limitations).

### Syncing subdirectories to match permission boundaries

By default Sourcegraph only supports repository-level permissions and does not match the granularity of the [Perforce protects file](https://www.perforce.com/manuals/cmdref/Content/CmdRef/p4_protect.html).

If you don't [activate file-level permissions](#file-level-permissions) you should sync subdirectories of a depot using the `depots` configuration that best describes the most concrete path of your permissions boundary.

For example, if your Perforce depot `//depot/Talkhouse` has different permissions for `//depot/Talkhouse/main-dev` and subdirectories `//depot/Talkhouse/rel1.0/front`, `//depot/Talkhouse/rel1.0/back` we recommend setting the following `depots`:

```json
{
  "depots": [
    "//depot/Talkhouse/main-dev/",
    "//depot/Talkhouse/rel1.0/front/",
    "//depot/Talkhouse/rel1.0/back/"
  ]
}
```

By configuring each subdirectory that has unique permissions, Sourcegraph is able to recognize and enforce permissions for the sub-directories. You can **NOT** define these permissions as:

```json
{
  "depots": [
    "//depot/Talkhouse/main-dev/",
    "//depot/Talkhouse/rel1.0/",
    "//depot/Talkhouse/rel1.0/back/"
  ]
}
```

Since that would override the permissions for the `//depot/Talkhouse/rel1.0/back` depot.

#### Wildcards

[File-level permissions](#file-level-permissions) can handle wildcards in the protects file.
If file-level permissions is not enabled, Sourcegraph provides limited support for `*` and `...` paths, so the workaround of [adding sub-folders as separate repositories](#syncing-subdirectories-to-match-permission-boundaries) for the paths that employ wildcards needs to be followed.

### File-level permissions

File-level permissions make the [syncing of subdirectories to match permission boundaries](#syncing-subdirectories-to-match-permission-boundaries) unnecessary.

To enable file-level permissions:

1. Enable [the feature in the site config](https://sourcegraph.com/github.com/sourcegraph/sourcegraph@2a716bd/-/blob/schema/site.schema.json?L227-249):

    ```json
    {
      "experimentalFeatures": {
        "subRepoPermissions": { "enabled": true }
      }
    }
    ```

1. Enable the feature in the code host configuration by adding `subRepoPermissions` to the `authorization` object:

    ```json
    {
      "authorization": {
        "subRepoPermissions": true
      }
    }
    ```

1. Save the configuration. Permissions will be synced in the background based on your [Perforce protects file](https://www.perforce.com/manuals/cmdref/Content/CmdRef/p4_protect.html).

### Notes about permissions

- Sourcegraph users are mapped to Perforce users based on their verified email addresses.
- As long as a user has been granted at least `Read` permissions in Perforce they will be able to view content in Sourcegraph.
- As a special case, commits in which a user does not have permissions to read any files are hidden. If a user can read a subset of files in a commit, only those files are shown.
- [The host field from protections are not supported](#known-issues-and-limitations).
- [file-level permissions must be disabled for Batch Changes to work](#known-issues-and-limitations).

## Configuration

<div markdown-func=jsonschemadoc jsonschemadoc:path="admin/external_service/perforce.schema.json">[View page on docs.sourcegraph.com](../../admin/external_service/perforce.schema.json) to see rendered content.</div>

## Known issues and limitations

We are actively working to significantly improve Sourcegraph's Perforce support. Please [file an issue](https://github.com/sourcegraph/sourcegraph/issues) to help us prioritize any specific improvements you'd like to see.

- Sourcegraph was initially built for Git repositories only, so it stores Perforce depots as Git repositories when syncing. Perforce concepts and languages are expressed in the UI, but under the hood, Git tools are used.
- The [host field](https://www.perforce.com/manuals/cmdref/Content/CmdRef/p4_protect.html#Form_Fields_..361) in protections is not supported.
- Batch Changes does not support [file-level permissions](#file-level-permissions) (also known as sub-repo permissions)
- Batch Changes does not handle the shelved changelist other than to query the Perforce server for its status.
- Permalinks with Changelist Id do not work yet
