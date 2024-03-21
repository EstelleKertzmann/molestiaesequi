<!-- *********************
  DO NOT EDIT THIS FILE
  It is a generated build output from Stardoc.
  Instead you must edit the .bzl file where the rules are declared,
  or possibly a markdown file next to the .bzl file
 ********************* -->

# Rollup rules for Bazel

The Rollup rules run the [rollup.js](https://rollupjs.org/) bundler with Bazel.

## Installation

Add the `@bazel/rollup` npm package to your `devDependencies` in `package.json`. (`rollup` itself should also be included in `devDependencies`, unless you plan on providing it via a custom target.)

### Installing with user-managed dependencies

If you didn't use the `yarn_install` or `npm_install` rule, you'll have to declare a rule in your root `BUILD.bazel` file to execute rollup:

```python
# Create a rollup rule to use in rollup_bundle#rollup_bin
# attribute when using user-managed dependencies
nodejs_binary(
    name = "rollup_bin",
    entry_point = "//:node_modules/rollup/bin/rollup",
    # Point bazel to your node_modules to find the entry point
    data = ["//:node_modules"],
)
```

## Usage

The `rollup_bundle` rule is used to invoke Rollup on some JavaScript inputs.
The API docs appear [below](#rollup_bundle).

Typical example:
```python
load("@npm//@bazel/rollup:index.bzl", "rollup_bundle")

rollup_bundle(
    name = "bundle",
    srcs = ["dependency.js"],
    entry_point = "input.js",
    config_file = "rollup.config.js",
)
```

Note that the command-line options set by Bazel override what appears in the rollup config file.
This means that typically a single `rollup.config.js` can contain settings for your whole repo,
and multiple `rollup_bundle` rules can share the configuration.

Thus, setting options that Bazel controls will have no effect, e.g.

```javascript
module.exports = {
    output: { file: 'this_is_ignored.js' },
}
```

### Output types

You must determine ahead of time whether Rollup will write a single file or a directory.
Rollup's CLI has the same behavior, forcing you to pick `--output.file` or `--output.dir`.

Writing a directory is used when you have dynamic imports which cause code-splitting, or if you
provide multiple entry points. Use the `output_dir` attribute to specify that you want a
directory output.

Each `rollup_bundle` rule produces only one output by running the rollup CLI a single time.
To get multiple output formats, you can wrap the rule with a macro or list comprehension, e.g.

```python
[
    rollup_bundle(
        name = "bundle.%s" % format,
        entry_point = "foo.js",
        format = format,
    )
    for format in [
        "cjs",
        "umd",
    ]
]
```

This will produce one output per requested format.

### Stamping

You can stamp the current version control info into the output by writing some code in your rollup config.
See the [stamping documentation](stamping).

By passing the `--stamp` option to Bazel, two additional input files will be readable by Rollup.

1. The variable `bazel_version_file` will point to `bazel-out/volatile-status.txt` which contains
statuses that change frequently; such changes do not cause a re-build of the rollup_bundle.
2. The variable `bazel_info_file` will point to `bazel-out/stable-status.txt` file which contains
statuses that stay the same; any changed values will cause rollup_bundle to rebuild.

Both `bazel_version_file` and `bazel_info_file` will be `undefined` if the build is run without `--stamp`.

> Note that under `--stamp`, only the bundle is re-built, but not the compilation steps that produced the inputs.
> This avoids a slow cascading re-build of a whole tree of actions.

To use these files, you write JS code in your `rollup.config.js` to read from the status files and parse the lines.
Each line is a space-separated key/value pair.

```javascript
/**
* The status files are expected to look like
* BUILD_SCM_HASH 83c699db39cfd74526cdf9bebb75aa6f122908bb
* BUILD_SCM_LOCAL_CHANGES true
* STABLE_BUILD_SCM_VERSION 6.0.0-beta.6+12.sha-83c699d.with-local-changes
* BUILD_TIMESTAMP 1520021990506
*
* Parsing regex is created based on Bazel's documentation describing the status file schema:
*   The key names can be anything but they may only use upper case letters and underscores. The
*   first space after the key name separates it from the value. The value is the rest of the line
*   (including additional whitespaces).
*
* @param {string} p the path to the status file
* @returns a two-dimensional array of key/value pairs
*/
function parseStatusFile(p) {
  if (!p) return [];
  const results = {};
  const statusFile = require('fs').readFileSync(p, {encoding: 'utf-8'});
  for (const match of `
${statusFile}`.matchAll(/^([A-Z_]+) (.*)/gm)) {
    // Lines which go unmatched define an index value of `0` and should be skipped.
    if (match.index === 0) {
      continue;
    }
    results[match[1]] = match[2];
  }
  return results;
}

// This undefined variable will be replaced with the full path during the build.
const statuses = parseStatusFile(bazel_version_file);
// Parse the stamp file produced by Bazel from the version control system
let version = '<unknown>';
// Don't assume BUILD_SCM_VERSION exists
if (statuses['BUILD_SCM_VERSION']) {
  version = 'v' + statuses['BUILD_SCM_VERSION'];
  if (DEBUG) {
    version += '_debug';
  }
}
```

### Debug and Opt builds

When you use `--compilation_mode=dbg`, Bazel produces a distinct output-tree in `bazel-out/[arch]-dbg/bin`.
Code in your `rollup.config.js` can look in the environment to detect if a debug build is being performed,
and include extra developer information in the bundle that you wouldn't normally ship to production.

Similarly, `--compilation_mode=opt` is Bazel's signal to perform extra optimizations.
You could use this value to perform extra production-only optimizations.

For example you could define a constant for enabling Debug:

```javascript
const DEBUG = process.env['COMPILATION_MODE'] === 'dbg';
```

and configure Rollup differently when `DEBUG` is `true` or `false`.

### Increasing Heap memory for rollup

The `rollup_bin` attribute allows you to customize the rollup.js program we execute,
so you can use `nodejs_binary` to construct your own.

> You can always call `bazel query --output=build [default rollup_bin]` to see what
> the default definition looks like, then copy-paste from there to be sure yours
> matches.

```python
nodejs_binary(
    name = "rollup_more_mem",
    data = ["@npm//rollup:rollup"],
    entry_point = "@npm//:node_modules/rollup/dist/bin/rollup",
    templated_args = [
        "--node_options=--max-old-space-size=<SOME_SIZE>",
    ],
)

rollup_bundle(
    ...
    rollup_bin = ":rollup_more_mem",
)
```


## rollup_bundle

**USAGE**

<pre>
rollup_bundle(<a href="#rollup_bundle-name">name</a>, <a href="#rollup_bundle-args">args</a>, <a href="#rollup_bundle-config_file">config_file</a>, <a href="#rollup_bundle-deps">deps</a>, <a href="#rollup_bundle-entry_point">entry_point</a>, <a href="#rollup_bundle-entry_points">entry_points</a>, <a href="#rollup_bundle-format">format</a>, <a href="#rollup_bundle-link_workspace_root">link_workspace_root</a>,
              <a href="#rollup_bundle-output_dir">output_dir</a>, <a href="#rollup_bundle-rollup_bin">rollup_bin</a>, <a href="#rollup_bundle-rollup_worker_bin">rollup_worker_bin</a>, <a href="#rollup_bundle-silent">silent</a>, <a href="#rollup_bundle-sourcemap">sourcemap</a>, <a href="#rollup_bundle-srcs">srcs</a>, <a href="#rollup_bundle-stamp">stamp</a>,
              <a href="#rollup_bundle-supports_workers">supports_workers</a>)
</pre>

Runs the rollup.js CLI under Bazel.

**ATTRIBUTES**


<h4 id="rollup_bundle-name">name</h4>

(*<a href="https://bazel.build/docs/build-ref.html#name">Name</a>, mandatory*): A unique name for this target.


<h4 id="rollup_bundle-args">args</h4>

(*List of strings*): Command line arguments to pass to Rollup. Can be used to override config file settings.

These argument passed on the command line before arguments that are added by the rule.
Run `bazel` with `--subcommands` to see what Rollup CLI command line was invoked.

See the <a href="https://rollupjs.org/guide/en/#command-line-flags">Rollup CLI docs</a> for a complete list of supported arguments.

Defaults to `[]`

<h4 id="rollup_bundle-config_file">config_file</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">Label</a>*): A `rollup.config.js` file

Passed to the `--config` option, see [the config doc](https://rollupjs.org/guide/en/#configuration-files)

If not set, a default basic Rollup config is used.

Defaults to `@npm//@bazel/rollup:rollup.config.js`

<h4 id="rollup_bundle-deps">deps</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">List of labels</a>*): Other libraries that are required by the code, or by the rollup.config.js

Defaults to `[]`

<h4 id="rollup_bundle-entry_point">entry_point</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">Label</a>*): The bundle's entry point (e.g. your main.js or app.js or index.js).

This is just a shortcut for the `entry_points` attribute with a single output chunk named the same as the rule.

For example, these are equivalent:

```python
rollup_bundle(
    name = "bundle",
    entry_point = "index.js",
)
```

```python
rollup_bundle(
    name = "bundle",
    entry_points = {
        "index.js": "bundle"
    }
)
```

If `rollup_bundle` is used on a `ts_library`, the `rollup_bundle` rule handles selecting the correct outputs from `ts_library`.
In this case, `entry_point` can be specified as the `.ts` file and `rollup_bundle` will handle the mapping to the `.mjs` output file.

For example:

```python
ts_library(
    name = "foo",
    srcs = [
        "foo.ts",
        "index.ts",
    ],
)

rollup_bundle(
    name = "bundle",
    deps = [ "foo" ],
    entry_point = "index.ts",
)
```

Defaults to `None`

<h4 id="rollup_bundle-entry_points">entry_points</h4>

(*<a href="https://bazel.build/docs/skylark/lib/dict.html">Dictionary: Label -> String</a>*): The bundle's entry points (e.g. your main.js or app.js or index.js).

Passed to the [`--input` option](https://github.com/rollup/rollup/blob/master/docs/999-big-list-of-options.md#input) in Rollup.

Keys in this dictionary are labels pointing to .js entry point files.
Values are the name to be given to the corresponding output chunk.

Either this attribute or `entry_point` must be specified, but not both.

Defaults to `{}`

<h4 id="rollup_bundle-format">format</h4>

(*String*): Specifies the format of the generated bundle. One of the following:

- `amd`: Asynchronous Module Definition, used with module loaders like RequireJS
- `cjs`: CommonJS, suitable for Node and other bundlers
- `esm`: Keep the bundle as an ES module file, suitable for other bundlers and inclusion as a `<script type=module>` tag in modern browsers
- `iife`: A self-executing function, suitable for inclusion as a `<script>` tag. (If you want to create a bundle for your application, you probably want to use this.)
- `umd`: Universal Module Definition, works as amd, cjs and iife all in one
- `system`: Native format of the SystemJS loader

Defaults to `"esm"`

<h4 id="rollup_bundle-link_workspace_root">link_workspace_root</h4>

(*Boolean*): Link the workspace root to the bin_dir to support absolute requires like 'my_wksp/path/to/file'.
If source files need to be required then they can be copied to the bin_dir with copy_to_bin.

Defaults to `False`

<h4 id="rollup_bundle-output_dir">output_dir</h4>

(*Boolean*): Whether to produce a directory output.

We will use the [`--output.dir` option](https://github.com/rollup/rollup/blob/master/docs/999-big-list-of-options.md#outputdir) in rollup
rather than `--output.file`.

If the program produces multiple chunks, you must specify this attribute.
Otherwise, the outputs are assumed to be a single file.

Defaults to `False`

<h4 id="rollup_bundle-rollup_bin">rollup_bin</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">Label</a>*): Target that executes the rollup binary

Defaults to `@npm//rollup/bin:rollup`

<h4 id="rollup_bundle-rollup_worker_bin">rollup_worker_bin</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">Label</a>*): Internal use only

Defaults to `@npm//@bazel/rollup/bin:rollup-worker`

<h4 id="rollup_bundle-silent">silent</h4>

(*Boolean*): Whether to execute the rollup binary with the --silent flag, defaults to False.

Using --silent can cause rollup to [ignore errors/warnings](https://github.com/rollup/rollup/blob/master/docs/999-big-list-of-options.md#onwarn) 
which are only surfaced via logging.  Since bazel expects printing nothing on success, setting silent to True
is a more Bazel-idiomatic experience, however could cause rollup to drop important warnings.

Defaults to `False`

<h4 id="rollup_bundle-sourcemap">sourcemap</h4>

(*String*): Whether to produce sourcemaps.

Passed to the [`--sourcemap` option](https://github.com/rollup/rollup/blob/master/docs/999-big-list-of-options.md#outputsourcemap") in Rollup

Defaults to `"inline"`

<h4 id="rollup_bundle-srcs">srcs</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">List of labels</a>*): Non-entry point JavaScript source files from the workspace.

You must not repeat file(s) passed to entry_point/entry_points.

Defaults to `[]`

<h4 id="rollup_bundle-stamp">stamp</h4>

(*<a href="https://bazel.build/docs/build-ref.html#labels">Label</a>*): Whether to encode build information into the output. Possible values:
    - `@rules_nodejs//nodejs/stamp:always`:
        Always stamp the build information into the output, even in [--nostamp][stamp] builds.
        This setting should be avoided, since it potentially causes cache misses remote caching for
        any downstream actions that depend on it.
    - `@rules_nodejs//nodejs/stamp:never`:
        Always replace build information by constant values. This gives good build result caching.
    - `@rules_nodejs//nodejs/stamp:use_stamp_flag`:
        Embedding of build information is controlled by the [--[no]stamp][stamp] flag.
        Stamped binaries are not rebuilt unless their dependencies change.
    [stamp]: https://docs.bazel.build/versions/main/user-manual.html#flag--stamp  The dependencies of this attribute must provide: StampSettingInfo


Defaults to `@rules_nodejs//nodejs/stamp:use_stamp_flag`

<h4 id="rollup_bundle-supports_workers">supports_workers</h4>

(*Boolean*): Experimental! Use only with caution.

Allows you to enable the Bazel Worker strategy for this library.
When enabled, this rule invokes the "rollup_worker_bin"
worker aware binary rather than "rollup_bin".

Defaults to `False`

