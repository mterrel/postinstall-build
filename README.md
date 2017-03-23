[![Travis Status][travis_img]][travis_site]
[![AppVeyor Status][appveyor_img]][appveyor_site]
[![npm Version][npm_img]][npm_site]
[![npm Downloads][downloads_img]][downloads_site]

# postinstall-build

Conditionally build in the `postinstall` hook without moving your
`devDependencies` to `dependencies`.

```shell
npm install postinstall-build --save
```

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Contents

- [What does it do?](#what-does-it-do)
- [Usage](#usage)
  - [Options](#options)
- [Motivation](#motivation)
- [Example](#example)
- [Caveats](#caveats)
  - [Bugs in npm](#bugs-in-npm)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## What does it do?

1. Check if your build artifacts exist.
2. If not, temporarily install `devDependencies` and build.
3. Clean up anything left behind… and that’s it!

## Usage

```shell
postinstall-build [options] <artifact> [command]
```

### Options

* `--silent`: Silence the build command’s stdout and stderr. This was the
  default behavior pre-1.0. Note that this may make debugging much more
  difficult if something goes wrong.
* `--script`: Run the given npm script from `package.json` instead of supplying
  a full build command. Specified like: `--script name` or `--script=name`.

If neither `command` nor `--script` is supplied, the build command defaults to
`npm run build`.

An `artifact` path is required. It should point to a file or directory that
will be generated by the build command. If the file already exists, the build
command won’t be run. If you want to always build (not recommended), just pass a
bogus file path.

Note that if your `command` contains arguments (and thus has spaces), you should
wrap it in escaped double quotes (`\"`) instead of single quotes for maximum
portability – Windows does not treat single-quoted strings as a single
parameter. (This is the case in any npm script regardless of `postinstall-build`
usage.)

## Motivation

Sometimes you want to install or depend on a package from someplace other than
npm – for example, from a `git` URL. If the package needs to be transpiled by
a tool like Babel, then this can be tricky: most people put their build step in
the `version` or `prepublish` hooks, and if you’re not installing from npm then
this step probably wasn’t run (unless the build artifacts are checked into
source).

One solution is to add a check to the package’s `postinstall` hook: if the
build artifacts don’t exist, then build! The annoying part is that this
necessitates having your build dependencies (like Babel or webpack) available –
in other words, they’d need to be production `dependencies` instead of
`devDependencies`, even though the module itself doesn’t `require` them (unlike
real dependencies, they’re only used in the build step). That means even
everyone installing from npm wastes time installing them, even though they
already have the build artifacts!

This helper fixes that. Just tell it where a build artifact is and what your
build step is, and it’ll do the rest. Used as intended, `postinstall-build`
should be in `dependencies`.

## Example

Here’s an example using Babel:

```json
{
  "scripts": {
    "build-lib": "babel --presets es2015 -d lib src",
    "postinstall": "postinstall-build lib \"npm run build-lib\""
  },
  "dependencies": {
    "postinstall-build": "^2.0.0"
  },
  "devDependencies": {
    "babel-cli": "^6.0.0",
    "babel-preset-es2015": "^6.0.0"
  }
}
```

The `postinstall-build` helper will check whether the first argument, `lib`,
exists. If not, it will run the second argument, `npm run build-lib`. Since
this probably requires your build dependencies (in this case Babel), it will
run `npm install --only=dev` first. When the build is done, it will run
`npm prune --production` to clean up. That’s it!

In this example the build command is another npm script. For convenience this
could also be written as `postinstall-build lib --script build-lib`.

## Caveats

### Bugs in Yarn

* **'your-package' is not in the npm registry.**

  Yarn will read your custom registry setting from `.npmrc`, but fails to
  communicate this via the `$npm_config_registry` environment variable. So any
  `npm` commands that were triggered by a Yarn install (like those run by
  `postinstall-build`) pick up Yarn‘s default `$npm_config_registry` setting
  instead of the one specified in `.npmrc`.
  
  For the time being you can solve this by adding a `.yarnrc` file alongside your
  `.npmrc`, which will cause `$npm_config_registry` to behave as expected.

### Bugs in npm

**I recommend using npm 3 or better, except for npm 4.1.x.**

There are several distinct bugs in npm itself that you may encounter when using
`postinstall-build` with npm 2. I have not been able to work around these nor
even reproduce them locally; they are especially prevalent on the combination
of Node 0.12, npm 2, and the Docker environment used by Travis. To the best of
my knowledge they are no fault of this package and are widely reported npm bugs.

* **extraneous packages**

  The `prune` command is broken in npm 4.1.x, and is unable to correctly prune
  `devDependencies`. Thus, when `postinstall-build` is finishing up, it leaves
  behind extraneous packages.

* **postinstall-build: not found**

  Sometimes npm triggers `postinstall` when a package’s dependencies aren’t
  actually available yet.

* **Callback called more than once.**

  npm has some faulty async code. This message comes from within the npm
  codebase and does not refer to any callbacks within `postinstall-build`.

* **ENOENT during npm prune**

  npm is probably trying to prune a file that was already removed or never
  existed. Seems to happen when there is a larger `devDependency` tree to prune.

* **ECONNRESET**

  npm has trouble making lots of connections to its own registry. You can use
  `npm config set fetch-retries 5` (for example) to work around this; using the
  non-HTTPS registry might also help.

[travis_img]: https://travis-ci.org/exogen/postinstall-build.svg
[travis_site]: https://travis-ci.org/exogen/postinstall-build
[appveyor_img]: https://ci.appveyor.com/api/projects/status/github/exogen/postinstall-build?svg=true
[appveyor_site]: https://ci.appveyor.com/project/exogen/postinstall-build
[npm_img]: https://img.shields.io/npm/v/postinstall-build.svg?label=npm+package
[npm_site]: https://www.npmjs.com/package/postinstall-build
[downloads_img]: https://img.shields.io/npm/dm/postinstall-build.svg
[downloads_site]: https://www.npmjs.com/package/postinstall-build
