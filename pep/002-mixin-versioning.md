---
title: "Mixin Versioning"
number: "NNN"
status: "provisional"
owner: "@carolynvs"
authors:
  - "@carolynvs"
  - "@sestegra"
created: "2021-02-21"
type: "feature"
---

## Abstract

Bundles should be able to specify not only the mixin used, but its version and source.
Porter should use that information to automatically download the mixins declared by the bundle.


## Motivation

Currently bundle authors manage the mixins used within a bundle by hand. 
They must manually install each mixin using homegrown scripts. 
Unfortunately the mixin cache is stored in PORTER_HOME, it is impossible to work on bundles that use different versions of the same mixin.
Porter really needs to manage the mixin cache and appropriately communicating with the correct mixin version targeted by the bundle.

* Mixin authors need to adjust their mixin declaration schema to allow for declaring the mixin version and release a new version of their mixin(s) supporting this proposal.
* Bundle authors should start defining the mixin version when they declare mixins in porter.yaml. If they have extra scripts that they were running before `porter build` to ensure the right versions are in the global mixin cache, that is no longer necessary.
* End users will now be able to see the exact version of the mixins used by a bundle.


## Rationale

<!--
The rationale fleshes out the specification by describing why particular design
decisions were made. It should describe alternate designs that were considered
and related work.

The rationale should provide evidence of consensus within the community and
discuss important objections or concerns raised during discussion.
-->
A lot of this design is driven by how mature tools in the ecosystem address this problem.
Since Porter's architecture is similar to Terraform, specifically Porter mixins and Terraform providers/modules, I have used Terraform's solutions to caching and locking the mixins used in a bundle to understand gaps in our own solution.

This feature was originally suggested at https://github.com/getporter/porter/issues/975 and by @sestegra in https://github.com/getporter/porter/discussions/1311.


## Specification

### Mixin Cache

First we need to alter the mixin cache structure in PORTER_HOME/mixins.
The new file structure allows for caching multiple versions of a mixin concurrently, and allows for disambiguation when the same mixin version is cached from different sources.
For example, two users can publish mixins with the same name and version and Porter should be able to distinguish between the two.

```
PORTER_HOME/
  mixins/
    SOURCE/ # May be nested directories, e.g. cdn.porter.sh/mixins/exec
      NAME/
        VERSION/
          NAME # client binary
          runtimes/
            NAME-GOOS-GOARCH # Runtime binary, e.g. aws-linux-amd64
```

❗️Security and reproducible builds for the mixin cache is something that must be addressed but is out-of-scope of this proposal.

#### Global Mixin Cache

The global mixin cache is located in PORTER_HOME/mixins.
Its purpose is to prevent repeatedly re-downloading mixins, which speeds up build times.

While the global mixin cache can be helpful for reproducible builds, for example in case a version is no longer accessible publicly, it is the responsibility of the user to ensure that by backing up the cache to a remote location and copying it locally back into the global mixin cache before building bundles.
Global cache management and reproducible builds are out-of-scope of this specification.

In CI systems, people are encouraged to preserve the global mixin cache to improve build times and limit bandwidth usage.


#### Bundle Mixin Cache

The local bundle mixin cache is located in a bundle directory under the .cnab/mixins directory.
This directory is managed by Porter and may be deleted at any time.

### Porter Lockfile

After Porter resolves the version of a mixin, it records that information in a new Porter lockfile, porter.lock.yaml, that is located next to porter.yaml.
This file ensures that once a mixin has been resolved, subsequent builds always use the same resolution.

```
mixins:
  exec:
    source: https://cdn.porter.sh/mixins/atom.xml
    url: https://cdn.porter.sh/mixins/exec/ # URL where the binaries are available
    version: v0.33.1
    binaries: # list all available binaries, not just what is used by the current client
      exec-darwin-amd64:
        digest: abc123
      exec-linux-amd64:
        digest: def456
```

When deciding if a mixin resolution can be used, Porter looks at the following:

* The mixin name and url must match.
* The resolved version must match the version range defined in porter.yaml

If the digest of the binaries do not match the lockfile, the build should return an error.


### Mixin Commands

We need to repurpose existing mixin commands now that we have a global mixins cache and a local bundle cache.
Let's review each mixin command and update its signature with a `--global` flag which indicates that the command should operate against the global cache. This flag defaults to false when the command is executed inside a bundle directory.


#### porter mixins search

This command does not operate on the cache so no changes are required.


#### porter mixins list

When global is false, list the mixins used by the bundle, taking into account the lockfile.

```
$ porter mixins list [--global=false]
Name         Version      Source
exec         v0.33.1      https://cdn.porter.sh/mixins/atom.xml
```

The json formatted output includes all the information available from the lockfile.

```json
[
 {
    "name": "exec",
    "source": "https://cdn.porter.sh/mixins/atom.xml",
    "url": "https://cdn.porter.sh/mixins/exec",
    "binaries": {
      "exec-darwin-amd64": {
        "digest": "abc123",
      },
      "exec-linux-amd64": {
        "digest": "def456",
      }
    },
    "version": "v0.33.1",
    "commit": "d433a66",
    "author": "Porter Authors"
  }
]
```

When global is true, list the mixins in the global cache:

```console
$ porter mixins list [--global=true]
Name         Versions
exec         12
helm         3
```

The json formatted output should list all versions in the global cache.


#### porter mixins install

When global is false, update the porter.yaml file to declare the specified mixin, install the mixin and cache it in the bundle mixin cache.
This mode is useful for automating the creation of bundles.

When global is true, install the mixin into the global bundle cache (current behavior).
This mode is useful for pre-populating the global mixin cache.


#### porter mixins upgrade

When global is false, bump the locked version of the mixin per the version range defined in the porter.yaml, and then update the bundle mixin cache.
This mode is useful for dependabot style CI integrations that automatically keep a bundle up-to-date.

When global is true, upgrade the specified mixin to the latest version in the global bundle cache.

#### porter mixins uninstall

❓ I am not sure if it makes sense to have keep this command? At the bundle level, removing a mixin doesn't seem to have a use case. They could just stop using it. However at the global cache level, what would really be useful is to clear the cache, or removing infrequently used mixin versions. I think cache management like that should be out-of-scope.


### Mixin Declaration

Bundle authors can optionally specify the source and version of the mixin that should be used with the bundle. Different versions/sources of a mixin cannot be used within the same bundle. 
When a source is not specified it defaults to the official porter mixin feed, and the version to latest (similar to how the `porter mixins install` command works).

```yaml
mixins:
    - arm:
        mixin:
            version: v1.2.3 # accepts a semver range, e.g. 1.x, 1.2.x
            source: https://example.com/feed.xml or https://github.com/getporter/terraform-mixins/downloads
        clientVersion: v1.2.3
    - terraform
```

❓ If possible, I want to avoid using complex semver ranges, e.g. ~1.2, ^1.1, >= 1.2 < 3.0.0 || >= 4.2.3, etc. The x notation is very easy to understand. Maybe we support whatever masterminds/semver supports but only document the x notation.

The mixin schema command should return information about the acceptable schema for the mixin's configuration.
Porter manages injecting the properties for `mixin -> version|source` when it assembles the final schema during `porter schema`.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "definitions": {
    "mixinConfig": {
      "type": "object",
      "properties": {
        "clientVersion": {
          "type": "string"
        }
      },
      "additionalProperties": false
    }    
  },
  "type": "object",
  "properties": {
    "install": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/installStep"
      }
    },
    "mixins": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/mixinConfig"
      }
    }
  }
}
```

### Mixin Resolution

A new command, `porter mixins download`, downloads the mixins used by the bundle to the local bundle mixin cache directory.

```
$ porter mixins download [--global=false]
```

1. Resolve the sources and versions of the mixins used by the bundle.
  If porter.lock.yaml exists, then any existing locked versions are used when the mixin name and source matches.
1. Identify if any required mixins are not in the global mixin cache, and then download them.
1. Copies the mixins into the local bundle cache using symbolic links to minimize storage requirements.
1. Updates the lock file with any newly resolved mixins.

Eventually people may want a command that allows pre-populating the global cache from a list of mixin sources.  However global cache management is out-of-scope of this proposal. If there is enough interest, we should follow-up with a proposal for general cache management (preventing cache size creep, warming up the cache, etc).


### Build

The `porter build` command implicitly calls `porter mixins download` before executing the build.
When the build command interacts with mixins, it should use the bundle mixin cache so that it does not need to re-resolve the mixins.

Information about the mixins used in the bundle should be extended to include not only the name but also the URL from which it was downloaded, its version and hashes of the client and runtime binaries.

```json
{
  "custom": {
    "sh.porter": {
      "mixins": {
        "exec": {
          "url": "https://cdn.porter.sh/mixins/exec/",
          "version": "v0.33.1",
          "binaries": {
            "exec-darwin-amd64": {
              "digest": "abc123"
            },
            "exec-linux-amd64": {
              "digest": "def456"
            }
          }
        }
      }
    }
  }
}
```

The manifest hash should be extended to include the lockfile as well so that builds are triggered properly when a mixin is updated.


### Verification

An end-user should be able to see which mixins are used by a bundle, where they came from and the version used.

```
$ porter explain --reference getporter/porter-hello:v0.1.0
Name: porter-hello
Version: v0.1.0
...

Mixins:
    Name    Version     URL
    exec    v0.33.1     https://cdn.porter.sh/mixins/arm
```

In the json output format, all available information about the mixins from the bundle.json custom metadata section should be returned.


## Implementation

<!--
After the PEP status is changed to implementable, when the PEP has been
implemented link to the pull request(s) here.
-->


## Backwards Compatibility

<!--
All PEPs that introduce backwards incompatibilities must include a section
describing these incompatibilities and their severity.  The PEP must explain how
to deal with these incompatibilities, possibly with defaulting or migrations.
PEP submissions without a sufficient backwards compatibility treatise may be
rejected outright.
-->


## Security Implications

<!--
If there are security concerns in relation to the PEP, those concerns should be
explicitly written out to make sure reviewers of the PEP are aware of them.
Mitigations should be included if possible.
-->


## Rejected Ideas

<!--
Throughout the discussion of a PEP, various ideas will be proposed which are not
accepted. Those rejected ideas should be recorded along with the reasoning as to
why they were rejected. This both helps record the thought process behind the
final version of the PEP as well as preventing people from bringing up the same
rejected idea again in subsequent discussions.
-->


## Open Questions

<!--
Before a PEP is implemented, questions can come up which warrant further discussion.
Those questions should be recorded here so people know that they are being
thought about but do not have a concrete resolution. This ensures all concerns
are addressed prior to accepting the PEP and reduces repeating prior discussion.
When possible, link the question to where it is being discussed, such as a
[forum] post/comment.
-->
