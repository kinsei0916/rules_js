# pnpm and rules_js

rules_js models dependency handling on [pnpm](https://pnpm.io/). Our design goal is to closely mimic pnpm's behavior.

Our story begins when some non-Bazel-specific tool (typically pnpm) performs dependency resolutions
and solves version constraints.
It also determines how the `node_modules` tree will be structured for runtime.
This information is encoded into a lockfile which is checked into the source repository.

The pnpm lockfile format includes all the information needed to define [`npm_import`](/docs/npm_import.md#npm_import) rules,
allowing Bazel's downloader to do the fetches. This info includes the integrity hash, as calculated by the package manager,
so that Bazel can guarantee supply-chain security.

Bazel will only fetch the packages which are required for the requested targets to be analyzed.
Thus it is performant to convert a very large `pnpm-lock.yaml` file without concern for
users needing to fetch many unnecessary packages. We have benchmarked this code with
800+ importers and ~15,000 npm packages to run in 3sec, when Bazel determines that an input changed.

While the [`npm_import`](/docs/npm_import.md#npm_import) rule can be used to bring individual packages into Bazel,
most users will want to import their entire lockfile.
The `npm_translate_lock` rule does this, and its operation is described below.
You may wish to read the [generated API documentation](/docs/npm_import.md#npm_translate_lock) as well.

## Using npm_translate_lock

In `WORKSPACE`, call the repository rule pointing to your `pnpm-lock.yaml` file:

```starlark
load("@aspect_rules_js//npm:npm_import.bzl", "npm_translate_lock")

# Uses the pnpm-lock.yaml file to automate creation of npm_import rules
npm_translate_lock(
    # Creates a new repository named "@npm" - you could choose any name you like
    name = "npm",
    pnpm_lock = "//:pnpm-lock.yaml",
    # Recommended attribute that also checks the .bazelignore file
    verify_node_modules_ignored = "//:.bazelignore",
)
```

You can immediately load from the generated `repositories.bzl` file in `WORKSPACE`.
This is similar to the
[`pip_parse`](https://github.com/bazelbuild/rules_python/blob/main/docs/pip.md#pip_parse)
rule in rules_python for example.
It has the advantage of also creating aliases for simpler dependencies that don't require
spelling out the version of the packages.

```starlark
# Following our example above, we named this "npm"
load("@npm//:repositories.bzl", "npm_repositories")

npm_repositories()
```

Note that you could call `npm_translate_lock` more than once, if you have more than one pnpm workspace in your Bazel workspace.

If you really don't want to rely on this being generated at runtime, we have experimental support
to check in the result instead. See [checked-in repositories.bzl](#checked-in-repositoriesbzl) below.

## Hoisting

The `node_modules` tree laid out by `rules_js` should be bug-for-bug compatible with the `node_modules` tree that
pnpm lays out, when [hoisting](https://pnpm.io/npmrc#hoist) is disabled.

To make the behavior outside of Bazel match, we recommend adding `hoist=false` to your `.npmrc`:

```shell
echo "hoist=false" >> .npmrc
```

This will prevent pnpm from creating a hidden `node_modules/.pnpm/node_modules` folder with hoisted
dependencies which allows packages to depend on "phantom" undeclared dependencies.
With hoisting disabled, most import/require failures (in type-checking or at runtime)
in 3rd party npm packages when using `rules_js` will be reproducible with pnpm outside of Bazel.

`rules_js` does not and will not support pnpm "phantom" [hoisting](https://pnpm.io/npmrc#hoist) which allows for
packages to depend on undeclared dependencies.
All dependencies between packages must be declared under `rules_js` in order to support lazy fetching and lazy linking of npm dependencies.

If a 3rd party npm package is relying on "phantom" dependencies to work, the recommended fix for `rules_js` is to
use [pnpm.packageExtensions](https://pnpm.io/package_json#pnpmpackageextensions) in your `package.json` to add the
missing `dependencies` or `peerDependencies`. For example,
https://github.com/aspect-build/rules_js/blob/a8c192eed0e553acb7000beee00c60d60a32ed82/package.json#L12.

NB: We plan to add support for the `.npmrc` `public-hoist-pattern` setting to `rules_js` in a future release.
For now, you can emulate public-hoist-pattern in `rules_js` using the `public_hoist_packages` attribute
of `npm_translate_lock`.

## Creating and updating the pnpm-lock.yaml file

### Manual (typical)

If your developers are fully converted to using pnpm, then they'll likely perform workflows like
adding new dependencies by running the pnpm tool in the source directory outside of Bazel.
This results in updates to the `pnpm-lock.yaml` file, and then Bazel naturally finds those updates
next time it reads the file.

### update_pnpm_lock

During a migration, you may have a legacy lockfile from another package manager.
You can use the `update_pnpm_lock` attribute of `npm_translate_lock` to have
Bazel manage the `pnpm-lock.yaml` file for you.
You might also choose this mode if you want changes like additions to `package.json` to be automatically
reflected in the lockfile, unlike a typical frontend developer workflow.

Use of `update_pnpm_lock` requires the `data` attribute be used as well.
This should include the `pnpm-workspace.yaml` file as well as all `package.json` files
in the pnpm workspace.
The pnpm lock file update will fail if `data` is missing any files required to run
`pnpm install --lockfile-only` or `pnpm import`.

> To list all local `package.json` files that pnpm needs to read, you can run
> `pnpm recursive ls --depth -1 --porcelain`.

A `.aspect/rules/external_repository_action_cache/npm_translate_lock_<hash>` file will be created
and used to determine when the `pnpm-lock.yaml` file should be updated.
This file should be checked into the source control along with the `pnpm-lock.yaml` file.

When the `pnpm-lock.yaml` file needs updating, `npm_translate_lock` will automatically:

-   run `pnpm import` if there is a `npm_package_lock` or `yarn_lock` attribute specified.
-   run `pnpm install --lockfile-only` otherwise.

To update the `pnpm-lock.yaml` file manually, either

-   [install pnpm](https://pnpm.io/installation) and run `pnpm install --lockfile-only` or `pnpm import`
-   use the Bazel-managed pnpm by running `bazel run -- @pnpm//:pnpm --dir $PWD install --lockfile-only` or `bazel run -- @pnpm//:pnpm --dir $PWD import`

If the `ASPECT_RULES_JS_FROZEN_PNPM_LOCK` environment variable is set and `update_pnpm_lock` is True,
the build will fail if the pnpm lock file needs updating.
It is recommended to set this environment variable on CI when `update_pnpm_lock` is True.

## Working with packages

### Patching via pnpm.patchedDependencies

Patches included in [pnpm.patchedDependencies](https://pnpm.io/next/package_json#pnpmpatcheddependencies) are automatically applied. These patches must be included in the `data` attribute of `npm_translate_lock`, for example:

```json
{
    ...
    "pnpm": {
        "patchedDependencies": {
            "fum@0.0.1": "patches/fum@0.0.1.patch"
        }
    }
}
```

```starlark
npm_translate_lock(
    ...
    data = [
        "//:patches/fum@0.0.1.patch",
    ],
)
```

### Patching via `patches` attribute

We recommend patching via [pnpm.patchedDependencies](#patching-via-pnpmpatcheddependencies) as above, but if you are importing
a yarn or npm lockfile and do not have this field in your package.json, you can apply additional
patches using the `patches` and `patch_args` attributes of `npm_translate_lock`.

These are designed to be similar to the same-named attributes of
[http_archive](https://bazel.build/rules/lib/repo/http#http_archive-patch_args).

Paths in patch files must be relative to the root of the package.
If the version is left out of the package name, the patch will be applied to every version of the npm package.

`patch_args` defaults to `-p0`, but `-p1` will usually be needed for patches generated by git.

In case multiple entries in `patches` match, the list of patches are additive.
(More specific matches are appended to previous matches.)
However if multiple entries in `patch_args` match, then the more specific name matches take precedence.

Patches in `patches` are applied after any patches included in `pnpm.patchedDependencies`.

For example,

```starlark
npm_translate_lock(
    ...
    patches = {
        "@foo/bar": ["//:patches/foo+bar.patch"],
        "fum@0.0.1": ["//:patches/fum@0.0.1.patch"],
    },
    patch_args = {
        "*": ["-p1"]
        "@foo/bar": ["-p0"]
        "fum@0.0.1": ["-p2"]
    },
)
```

### Lifecycles

npm packages have "lifecycle scripts" such as `postinstall` which are documented here:
<https://docs.npmjs.com/cli/v9/using-npm/scripts#life-cycle-scripts>

We refer to these as "lifecycle hooks".

> You can disable this feature completely by setting all packages to have no hooks, using
> `lifecycle_hooks = { "*": [] }` in `npm_translate_lock`.

Because rules_js models the execution of these hooks as build actions, rather than repository rules,
the result can be stored in the remote cache and shared between developers.
Typically these actions are not run in Bazel's action sandbox because of the overhead of setting up and tearing down the sandboxes.

In addition to sandboxing, Bazel supports other `execution_requirements` for actions,
in the attribute of <https://bazel.build/rules/lib/actions#run>.
You can have control over these using the `lifecycle_hooks_execution_requirements` attribute of `npm_translate_lock`.

Some hooks may fail to run under rules_js, and you don't care to run them.
You can use the `lifecycle_hooks_exclude` attribute of `npm_translate_lock` to turn them off for a package,
which is equivalent to setting the `lifecycle_hooks` to an empty list for that package.

You can set environment variables for hook build actions using the `lifecycle_hooks_envs` attribute of `npm_translate_lock`.

In case there are multiple matches, some attributes are additive.
(More specific matches are appended to previous matches.)
Other attributes have specificity: the most specific match wins and the others are ignored.

| attribute                              | behavior    |
| -------------------------------------- | ----------- |
| lifecycle_hooks                        | specificity |
| lifecycle_hooks_envs                   | additive    |
| lifecycle_hooks_execution_requirements | specificity |

Here's a complete example of managing lifecycles:

```starlark
npm_translate_lock(
    ...
    lifecycle_hooks = {
        # These three values are the default if lifecycle_hooks was absent
        # do not sort
        "*": [
            "preinstall",
            "install",
            "postinstall",
        ],
        # This package comes from a git url so prepare has to run to compile some things
        "@kubernetes/client-node": ["prepare"],
        # Disable install and preinstall for this package, maybe they are broken
        "fum@0.0.1": ["postinstall"],
    },
    lifecycle_hooks_envs: {
        # Set some values for all hook actions
        "*": [
            "GLOBAL_KEY1=value1",
            "GLOBAL_KEY2=value2",
        ],
        # ... but override for this package
        "@foo/bar": [
            "GLOBAL_KEY2=",
            "PREBULT_BINARY=http://downloadurl",
        ],
    },
    lifecycle_hooks_execution_requirements = {
        # This is the default if lifecycle_hooks_execution_requirements was absent
        "*":         ["no-sandbox"],
        # Omit no-sandbox for this package, maybe it relies on sandboxing to succeed
        "@foo/bar":  [],
        # This one is broken in remote execution for whatever reason
        "fum@0.0.1": ["no-sandbox", "no-remote-exec"],
    }
)
```

In this example:

-   Only the `prepare` lifecycle hook will be run for the `@kubernetes/client-node` npm package,
    only the `postinstall` will be run for `fum` at version 0.0.1,
    and the default hooks are run for remaining packages.
-   `@foo/bar` lifecycle hooks will run with Bazel's sandbox enabled, with an effective environment:
    -   `GLOBAL_KEY1=value1`
    -   `GLOBAL_KEY2=`
    -   `PREBULT_BINARY=http://downloadurl`
-   `fum` at version 0.0.1 has remote execution disabled. Like other packages aside from `@foo/bar`
    the action sandbox is disabled for performance.

## Checked-in repositories.bzl

This usage is experimental and difficult to get right! Read on with caution.

You can check in the `repositories.bzl` file to version control, and load that instead.

This makes it easier to ship a ruleset that has its own npm dependencies, as users don't
have to install those dependencies. It also avoids eager-evaluation of `npm_translate_lock`
for builds that don't need it.
This is similar to the [`update-repos`](https://github.com/bazelbuild/bazel-gazelle#update-repos)
approach from bazel-gazelle.

The tradeoffs are similar to
[this rules_python thread](https://github.com/bazelbuild/rules_python/issues/608).

In a BUILD file, use a rule like
[write_source_files](https://github.com/aspect-build/bazel-lib/blob/main/docs/write_source_files.md)
to copy the generated file to the repo and test that it stays updated:

```starlark
write_source_files(
    name = "update_repos",
    files = {
        "repositories.bzl": "@npm//:repositories.bzl",
    },
)
```

Then in `WORKSPACE`, load from that checked-in copy or instruct your users to do so.
