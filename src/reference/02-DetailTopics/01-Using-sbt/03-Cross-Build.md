---
out: Cross-Build.html
---

  [Command-Line-Reference]: Command-Line-Reference.html
  [Publishing]: Publishing.html
  [Cross-Build-Plugins]: Cross-Build-Plugins.html
  [playframework4520]: https://github.com/playframework/playframework/pull/4520

Cross-building
--------------

### Introduction

Different versions of Scala can be binary incompatible, despite
maintaining source compatibility. This page describes how to use `sbt`
to build and publish your project against multiple versions of Scala and
how to use libraries that have done the same.

### Publishing conventions

The underlying mechanism used to indicate which version of Scala a
library was compiled against is to append `_<scala-version>` to the
library's name. For Scala 2.10.0 and later, the binary version is used.
For example, `dispatch-core_2.10` when compiled against
2.10.0, 2.10.1 or any 2.10.x version. This fairly simple approach
allows interoperability with users of Maven, Ant and other build tools.

The rest of this page describes how `sbt` handles this for you as part
of cross-building.

### Using cross-built libraries

To use a library built against multiple versions of Scala, double the
first `%` in an inline dependency to be `%%`. This tells `sbt` that it
should append the current version of Scala being used to build the
library to the dependency's name. For example:

```scala
libraryDependencies += "net.databinder.dispatch" %% "dispatch-core" % "0.13.3"
```
A nearly equivalent, manual alternative for a fixed version of Scala is:

```scala
libraryDependencies += "net.databinder.dispatch" % "dispatch-core_2.12" % "0.13.3"
```

### Cross building a project

Define the versions of Scala to build against in the
`crossScalaVersions` setting. Versions of Scala 2.10.2 or later are
allowed. For example, in a `.sbt` build definition:

```scala
lazy val scala212 = "$example_scala_version$"
lazy val scala211 = "$example_scala211$"
lazy val supportedScalaVersions = List(scala212, scala211)

ThisBuild / organization := "com.example"
ThisBuild / version      := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := scala212

lazy val root = (project in file("."))
  .aggregate(util, core)
  .settings(
    // crossScalaVersions must be set to Nil on the aggregating project
    crossScalaVersions := Nil,
    publish / skip := true
  )

lazy val core = (project in file("core"))
  .settings(
    crossScalaVersions := supportedScalaVersions,
    // other settings
  )

lazy val util = (project in file("util"))
  .settings(
    crossScalaVersions := supportedScalaVersions,
    // other settings
  )
```

**Note**: `crossScalaVersions` must be set to `Nil` on the root project to avoid double publishing.

To build against all versions listed in `crossScalaVersions`, prefix
the action to run with `+`. For example:

```
> + test
```

A typical way to use this feature is to do development on a single Scala
version (no `+` prefix) and then cross-build (using `+`) occasionally
and when releasing.

#### Note about sbt-release

sbt-release implemented cross building support by copy-pasting sbt 0.13's `+` implementation,
so at least as of sbt-release 1.0.10, it does not work correctly with sbt 1.x's cross building,
which was prototyped originally as sbt-doge.

To cross publish using sbt-release with sbt 1.x, use the following workaround:

```scala
ThisBuild / organization := "com.example"
ThisBuild / version      := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := scala212

import ReleaseTransformations._
lazy val root = (project in file("."))
  .aggregate(util, core)
  .settings(
    // crossScalaVersions must be set to Nil on the aggregating project
    crossScalaVersions := Nil,
    publish / skip := true,

    // don't use sbt-release's cross facility
    releaseCrossBuild := false,
    releaseProcess := Seq[ReleaseStep](
      checkSnapshotDependencies,
      inquireVersions,
      runClean,
      releaseStepCommandAndRemaining("+test"),
      setReleaseVersion,
      commitReleaseVersion,
      tagRelease,
      releaseStepCommandAndRemaining("+publishSigned"),
      setNextVersion,
      commitNextVersion,
      pushChanges
    )
  )
```

This will then use the real cross (`+`) implementation for testing and publishing.
Credit for this technique goes to James Roper at [playframework#4520][playframework4520] and later inventing `releaseStepCommandAndRemaining`.

#### Cross building with a Java project

A special care must be taken when cross building involves pure Java project.
Let's say in the following example, `network` is a Java project, and `core` is
a Scala project that depends on `network`.

```scala
lazy val scala212 = "$example_scala_version$"
lazy val scala211 = "$example_scala211$"
lazy val supportedScalaVersions = List(scala212, scala211)

ThisBuild / organization := "com.example"
ThisBuild / version      := "0.1.0-SNAPSHOT"
ThisBuild / scalaVersion := scala212

lazy val root = (project in file("."))
  .aggregate(network, core)
  .settings(
    // crossScalaVersions must be set to Nil on the aggregating project
    crossScalaVersions := Nil,
    publish / skip := false
  )

// example Java project
lazy val network = (project in file("network"))
  .settings(
    // set to exactly one Scala version
    crossScalaVersions := List(scala212),
    crossPaths := false,
    autoScalaLibrary := false,
    // other settings
  )

lazy val core = (project in file("core"))
  .dependsOn(network)
  .settings(
    crossScalaVersions := supportedScalaVersions,
    // other settings
  )
```

1. `crossScalaVersions` must be set to `Nil` on the aggregating projects such as the root.
2. Java subprojects should have exactly one Scala version in `crossScalaVersions` to avoid double publishing, typically `scala212`.
3. Scala subprojects can have multiple Scala versions in `crossScalaVersions`, but must avoid aggregating Java subprojects.

#### Switching Scala version

You can use `++ <version> [command]` to temporarily switch the Scala version currently
being used to build the subprojects given that `<version>` is listed in their `crossScalaVersions`.

For example:

```
> ++ $example_scala_version$
[info] Setting version to $example_scala_version$
> ++ $example_scala211$
[info] Setting version to $example_scala211$
> compile
```

`<version>` should be either a version for Scala published to a repository or
the path to a Scala home directory, as in `++ /path/to/scala/home`.
See [Command Line Reference][Command-Line-Reference] for details.

When a `[command]` is passed in to `++`, it will execute the command
on the subprojects that supports the given `<version>`.

For example:

```
> ++ $example_scala211$ -v test
[info] Setting Scala version to $example_scala211$ on 1 projects.
[info] Switching Scala version on:
[info]     core ($example_scala_version$, $example_scala211$)
[info] Excluding projects:
[info]   * root ()
[info]     network ($example_scala_version$)
[info] Reapplying settings...
[info] Set current project to core (in build file:/Users/xxx/hello/)
```

Sometimes you might want to force the Scala version switch regardless of the `crossScalaVersions` values.
You can use `++ <version>!` with exclamation mark for that.

For example:

```
> ++ 2.13.0-M5! -v
[info] Forcing Scala version to 2.13.0-M5 on all projects.
[info] Switching Scala version on:
[info]   * root ()
[info]     core ($example_scala_version$, $example_scala211$)
[info]     network ($example_scala_version$)
```

#### Cross publishing

The ultimate purpose of `+` is to cross-publish your
project. That is, by doing:

```
> + publishSigned
```

you make your project available to users for different versions of
Scala. See [Publishing][Publishing] for more details on publishing your project.

In order to make this process as quick as possible, different output and
managed dependency directories are used for different versions of Scala.
For example, when building against Scala 2.12.7,

-   `./target/` becomes `./target/scala_2.12/`
-   `./lib_managed/` becomes `./lib_managed/scala_2.12/`

Packaged jars, wars, and other artifacts have `_<scala-version>`
appended to the normal artifact ID as mentioned in the Publishing
Conventions section above.

This means that the outputs of each build against each version of Scala
are independent of the others. `sbt` will resolve your dependencies for
each version separately. This way, for example, you get the version of
Dispatch compiled against 2.11 for your 2.11.x build, the version
compiled against 2.12 for your 2.12.x builds, and so on. You can have
fine-grained control over the behavior for different Scala versions
by using the `cross` method on `ModuleID` These are equivalent:

```scala
"a" % "b" % "1.0"
"a" % "b" % "1.0" cross CrossVersion.Disabled
```

These are equivalent:

```scala
"a" %% "b" % "1.0"
"a" % "b" % "1.0" cross CrossVersion.binary
```

This overrides the defaults to always use the full Scala version instead
of the binary Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.full
```

`CrossVersion.patch` sits between `CrossVersion.binary` and `CrossVersion.full`
in that it strips off any trailing `-bin-...` suffix which is used to
distinguish varaint but binary compatible Scala toolchain builds.

```scala
"a" % "b" % "1.0" cross CrossVersion.patch
```

This uses a custom function to determine the Scala version to use based
on the binary Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.binaryMapped {
  case "2.9.1" => "2.9.0" // remember that pre-2.10, binary=full
  case "2.10" => "2.10.0" // useful if a%b was released with the old style
  case x => x
}
```

This uses a custom function to determine the Scala version to use based
on the full Scala version:

```scala
"a" % "b" % "1.0" cross CrossVersion.fullMapped {
  case "2.9.1" => "2.9.0"
  case x => x
}
```

A custom function is mainly used when cross-building and a dependency
isn't available for all Scala versions or it uses a different convention
than the default.
