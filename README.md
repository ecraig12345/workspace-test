Testing yarn workspace `nohoist` settings.

# Notes: yarn `resolutions` and `nohoist`

Supposing you have this package structure in a monorepo using Yarn v1 workspaces (based on [microsoft/fluentui](https://github.com/microsoft/fluentui)):
```
packages/
  pkg1/         @fluentui/pkg1
  pkg2/         @fluentui/pkg2
  weird-pkg/    @fluentui/weird-pkg
scripts/
```
`weird-pkg` is so called because it needs to have some different dependency versions (tooling in our case) than the rest of the monorepo. `scripts` *(not currently in this repo)* hosts the tooling for the repo.

### nohoist

Yarn workspaces have a [`nohoist` setting](https://classic.yarnpkg.com/blog/2018/02/15/nohoist/) which can be used to prevent specific dependencies from being hoisted to the root.

Some examples of preventing hoisting of a specific dep of `weird-pkg`:

```jsonc
{
  "workspaces": {
    "nohoist": [
      // Don't hoist weird-pkg's foo itself 
      "@fluentui/weird-pkg/foo",
      
      // Don't hoist any deps of weird-pkg's foo
      "@fluentui/weird-pkg/foo/**",

      // Don't hoist any nested instances of foo under weird-pkg
      // (warning: may result in a lot of copies of foo nested under weird-pkg;
      // does not affect location of weird-pkg's direct dep on foo)
      "@fluentui/weird-pkg/**/foo",

      // In theory this should prevent hoisting of ALL immediate and nested deps of weird-pkg
      // (haven't tested it)
      "@fluentui/weird-pkg/**"
    ]
  }
}
```

### resolutions

Yarn's [resolutions](https://classic.yarnpkg.com/en/docs/selective-version-resolutions/) feature forces dependencies to resolve to a specific version using `resolutions` in the root `package.json`.

One notable difference from `nohoist` is that if you want to change resolutions of deps under a particular workspace package, you need to start the path with `**/`, e.g. `**/@fluentui/weird-pkg`.

```jsonc
{
  "resolutions": {
    // Force ALL deps on foo (explicit and nested) to resolve to 1.2.3 (regardless of semver)
    "foo": "1.2.3",

    // Suppose foo is a dep of various workspace packages.
    // In ALL instances, force foo's dep on bar to 1.2.3 (regardless of semver).
    "**/foo/bar": "1.2.3",
    // This way doesn't work if it violates semver.
    // "foo/bar": "1.2.3"

    // Force weird-pkg's NESTED deps of foo to resolve to 1.2.3 (regardless of semver).
    // If weird-pkg itself has a dep on foo, it will ONLY be forced to 1.2.3 if that matches semver.
    "**/@fluentui/weird-pkg/**/foo": "1.2.3",

    // Not very useful (you should probably just change weird-pkg's foo dep instead):
    // Force only weird-pkg's dep on foo to resolve to 1.2.3, ONLY if it matches semver.
    // Does not affect nested deps on foo.
    "**/@fluentui/weird-pkg/foo": "1.2.3",
  }
}
```

### node_modules/.bin linking

Yarn's behavior for which version of a dependency it decides to link in `<workspace root>/node_modules/.bin` is rather odd in some cases... 

In this example we'll use the dep on `typescript`. Most of the monorepo uses one typescript version, but `weird-pkg` needs to use a different version.

If the root `package.json` specifies a version of `typescript`, it will link that version (good).

Otherwise...behavior is unclear and potentially varies between computers. :scream:

In microsoft/fluentui we had [an issue](https://github.com/microsoft/fluentui/issues/14371) where certain packages were using the wrong version of typescript. I couldn't repro this, but other team members could.

Eventually (while working on some additional config/dep changes) I realized I could try `ls -l node_modules/.bin/tsc`. At the workspace root and with my local changes, this gave the result: `../../packages/weird-pkg/node_modules/@microsoft/api-extractor/node_modules/typescript/bin/tsc`

... (more details later maybe) ...

Workaround: to ensure the expected `bin` version of a dep is used, either:
- Specify it in the root `package.json`
- Specify it in EVERY package where the `bin` is used

### More readable `yarn why`

A terrible script for making `yarn why`'s output easier to read.

`whyyy () { yarn why "$*" | sed -E '/Disk size/d;/shared dependencies/d;/\/4]/d;s/^info /   /g;s/Hoisted from //g;s/"//g;s/This module exists because /- /g;s/ depends on it\.?//g;/Done in/d;/Reasons this/d' | perl -p -e 's/=> Found /\n/' && echo; };`
