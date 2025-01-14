---
id: OrganizeImports
title: OrganizeImports
---

OrganizeImports was originally a [community rule](https://github.com/liancheng/scalafix-organize-imports),
created and maintained by [Cheng Lian](https://github.com/liancheng).
Considering its [popularity, maturity and the lack of bandwidth from the
author](https://github.com/liancheng/scalafix-organize-imports/discussions/215),
it is now a built-in rule, since Scalafix 0.11.0.

Getting started
---------------

### For IntelliJ Scala plugin users

`OrganizeImports` allows you to specify a preset style via the [`preset`
option](#preset). To make it easier to add `OrganizeImports` into
existing Scala projects built using the IntelliJ Scala plugin,
`OrganizeImports` provides a preset style compatible with the default
configuration of the IntelliJ Scala import optimizer. Please check the
[`INTELLIJ_2020_3`](#intellij_2020_3) preset style for more details.

### Source formatting tools

The `OrganizeImports` rule respects source-formatting tools like
[Scalafmt](https://scalameta.org/scalafmt/). If an import statement is
already organized according to the configuration, its original source
level format is preserved. Therefore, in an sbt project, if you run the
following command sequence:

```
sbt> scalafixAll
...
sbt> scalafmtAll
...
sbt> scalafixAll --check
...
```

Assuming that the first two commands run successfully, the last
`scalafixAll --check` command should not fail even if some import
statements are reformatted by the `scalafmtAll` command.

However, you should make sure that the source-formatting tools you use
do not rewrite import statements in ways that conflict with
`OrganizeImports`. For example, when using Scalafmt together with
`OrganizeImports`, the `ExpandImportSelectors`, `SortImports`, and
`AsciiSortImports` rewriting rules should not be used.

### Scala 3

Known limitations:

1.  The [`removeUnused`](OrganizeImports.md#removeunused) option must be
    explicitly set to `false` - the rule currently doesn’t remove unused
    imports as it is currently not supported by the compiler.

2.  Usage of [deprecated package
    objects](http://dotty.epfl.ch/docs/reference/dropped-features/package-objects.html)
    may result in incorrect imports.

3.  The
    [`groupExplicitlyImportedImplicitsSeparately`](OrganizeImports.md#groupexplicitlyimportedimplicitsseparately)
    option has no effect.

Configuration
-------------

> Please do NOT use the [`RemoveUnused.imports`](RemoveUnused.md) together with
> `OrganizeImports` to remove unused imports. You may end up with broken code!
> It is still safe to use `RemoveUnused` to remove unused private members or
> local definitions, though.
> 
> Scalafix rewrites source files by applying patches generated by invoked
> rules. Each rule generates a patch based on the *original* text of the
> source files. When two patches generated by different rules conflict
> with each other, Scalafix is not able to reconcile the conflicts, and
> may produce broken code. It is very likely to happen when `RemoveUnused`
> and `OrganizeImports` are used together, since both rules rewrite import
> statements.
> 
> By default, `OrganizeImports` already removes unused imports for you
> (see the [`removeUnused`](OrganizeImports.md#removeunused) option). It locates unused
> imports via compilation diagnostics, which is exactly how `RemoveUnused`
> does it. This mechanism works well in most cases, unless there are new
> unused imports generated while organizing imports, which is possible
> when the [`expandRelative`](OrganizeImports.md#expandrelative) option is set to true. For
> now, the only reliable workaround for this edge case is to run Scalafix
> with `OrganizeImports` twice.

```scala mdoc:passthrough
import scalafix.internal.rule._
import scalafix.website._
```

```scala mdoc:passthrough
println(
  defaults(
    "OrganizeImports",
    flat(OrganizeImportsConfig.default)
  )
)
```

`blankLines`
------------

Configures whether blank lines between adjacent import groups are
automatically or manually inserted. This option is used together with
the [`---` blank line markers](OrganizeImports.md#a-blank-line-marker).

### Value type

Enum: `Auto | Manual`

#### `Auto`
A blank line is automatically inserted between adjacent import groups.
All blank line markers (`---`) configured in the [`groups`
option](#groups) are ignored.

#### `Manual`
A blank line is inserted at all the positions where blank line markers
appear in the [`groups` option](#groups).

The following two configurations are equivalent:

```conf
OrganizeImports {
  blankLines = Auto
  groups = [
    "re:javax?\\."
    "scala."
    "*"
  ]
}
```

```conf
OrganizeImports {
  blankLines = Manual
  groups = [
    "re:javax?\\."
    "---"
    "scala."
    "---"
    "*"
  ]
}
```

### Default value

`Auto`

### Examples

#### `Auto`  

```conf
OrganizeImports {
  blankLines = Auto
  groups = [
    "re:javax?\\."
    "scala."
    "*"
  ]
}
```

Before:

```scala
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext

import sun.misc.BASE64Encoder
```

#### `Manual`  

```conf
OrganizeImports {
  blankLines = Manual
  groups = [
    "re:javax?\\."
    "scala."
    "---"
    "*"
  ]
}
```

Before:

```scala
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated
import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext

import sun.misc.BASE64Encoder
```

`coalesceToWildcardImportThreshold`
-----------------------------------

When the number of imported names exceeds a certain threshold, coalesce
them into a wildcard import. Renames and unimports are left untouched.

> Having this feature in `OrganizeImports` is mostly for feature parity
> with the IntelliJ IDEA Scala import optimizer, but coalescing grouped
> import selectors into a wildcard import may introduce *compilation
> errors*!
> 
> Here is an example to illustrate the risk. The following snippet
> compiles successfully:
> 
> ```scala
> import scala.collection.immutable._
> import scala.collection.mutable.{ArrayBuffer, Map, Set}
> 
> object Example {
>   val m: Map[Int, Int] = ???
> }
> ```
> 
> The type of `Example.m` above is not ambiguous because the mutable `Map`
> explicitly imported in the second import takes higher precedence than
> the immutable `Map` imported via wildcard in the first import.
> 
> However, if we coalesce the grouped imports in the second import
> statement into a wildcard, there will be a compilation error:
> 
> ```scala
> import scala.collection.immutable._
> import scala.collection.mutable._
> 
> object Example {
>   val m: Map[Int, Int] = ???
> }
> ```
> 
> This is because the type of `Example.m` becomes ambiguous now since both
> the mutable and immutable `Map` are imported via a wildcard and have the
> same precedence.

### Value type

Integer. Not setting it or setting it to `null` disables this feature.

### Default value

`null`

### Examples

```conf
OrganizeImports {
  groupedImports = Keep
  coalesceToWildcardImportThreshold = 3
}
```

Before:

```scala
import scala.collection.immutable.{Seq, Map, Vector, Set}
import scala.collection.immutable.{Seq, Map, Vector}
import scala.collection.immutable.{Seq, Map, Vector => Vec, Set, Stream}
import scala.collection.immutable.{Seq, Map, Vector => _, Set, Stream}
```

After:

```scala
import scala.collection.immutable._
import scala.collection.immutable.{Map, Seq, Vector}
import scala.collection.immutable.{Vector => Vec, _}
import scala.collection.immutable.{Vector => _, _}
```

`expandRelative`
----------------

Expand relative imports into fully-qualified one.

> Expanding relative imports may introduce new unused imports. For
> instance, relative imports in the following snippet
> 
> ```scala
> import scala.util
> import util.control
> import control.NonFatal
> ```
> 
> are expanded into
> 
> ```scala
> import scala.util
> import scala.util.control
> import scala.util.control.NonFatal
> ```
> 
> If neither `scala.util` nor `scala.util.control` is referenced anywhere
> after the expansion, they become unused imports.
> 
> Unfortunately, these newly introduced unused imports cannot be removed
> by setting `removeUnused` to `true`. Please refer to the
> [`removeUnused`](OrganizeImports.md#removeunused) option for more details.

### Value type

Boolean

### Default value

`false`

### Examples

```conf
OrganizeImports {
  expandRelative = true
  groups = ["re:javax?\\.", "scala.", "*"]
}
```

Before:

```scala
import scala.util
import util.control
import control.NonFatal
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext
import scala.util
import scala.util.control
import scala.util.control.NonFatal

import sun.misc.BASE64Encoder
```

`groupExplicitlyImportedImplicitsSeparately`
--------------------------------------------

This option provides a workaround to a subtle and rarely seen
correctness issue related to explicitly imported implicit names.

The following snippet helps illustrate the problem:

```scala
package a

import c._
import b.i

object b { implicit def i: Int = 1 }
object c { implicit def i: Int = 2 }

object Imports {
  def f()(implicit i: Int) = println(1)
  def main() = f()
}
```

The above snippet compiles successfully and outputs `1`, because the
explicitly imported implicit value `b.i` overrides `c.i`, which is made
available via a wildcard import. However, if we reorder the two imports
into:

```scala
import b.i
import c._
```

The Scala compiler starts complaining:

```
error: could not find implicit value for parameter i: Int
  def main() = f()
                ^
```

This behavior could be due to a Scala compiler bug since [the Scala
language
specification](https://scala-lang.org/files/archive/spec/2.13/02-identifiers-names-and-scopes.html)
requires that explicitly imported names should have higher precedence
than names made available via a wildcard.

Unfortunately, Scalafix is not able to surgically identify conflicting
implicit values behind a wildcard import. In order to guarantee
correctness in all cases, when the
`groupExplicitlyImportedImplicitsSeparately` option is set to `true`,
all explicitly imported implicit names are moved into the trailing
order-preserving import group together with relative imports, if any
(see the [trailing order-preserving import
group](OrganizeImports.md#groups) section for more
details).

> In general, order-sensitive imports are fragile, and can easily be
> broken by either human collaborators or tools (e.g., the IntelliJ IDEA
> Scala import optimizer does not handle this case correctly). They should
> be eliminated whenever possible. This option is mostly useful when you
> are dealing with a large trunk of legacy codebase, and you want to
> minimize manual intervention and guarantee correctness in all cases.

> The `groupExplicitlyImportedImplicitsSeparately` option has currently no
> effect on source files compiled with Scala 3, as the [compiler does not
> expose full signature
> information](https://github.com/lampepfl/dotty/issues/12766), preventing
> the rule to identify imported implicits.

### Value type

Boolean

### Default value

`false`

Rationale:

1.  Although setting it to `true` avoids the aforementioned correctness
    issue, the result is unintuitive and confusing for many users since
    it looks like the `groups` option is not respected.

    E.g., why my `scala.concurrent.ExecutionContext.Implicits.global`
    import is moved to a separate group even if I have a `scala.` group
    defined in the `groups` option?

2.  The concerned correctness issue is rarely seen in real life. When it
    really happens, it is usually a sign of bad coding style, and you
    may want to tweak your imports to eliminate the root cause.

### Examples

```conf
OrganizeImports {
  groups = ["scala.", "*"]
  groupExplicitlyImportedImplicitsSeparately = true // not supported in Scala 3
}
```

Before:

```scala
import org.apache.spark.SparkContext
import org.apache.spark.RDD
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.Buffer
import scala.concurrent.ExecutionContext.Implicits.global
import scala.sys.process.stringToProcess
```

After:
```scala
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.Buffer

import org.apache.spark.RDD
import org.apache.spark.SparkContext

import scala.concurrent.ExecutionContext.Implicits.global
import scala.sys.process.stringToProcess
```

`groupedImports`
----------------

Configure how to handle grouped imports.

### Value type

Enum: `Explode | Merge | AggressiveMerge | Keep`

#### `Explode`  
Explode grouped imports into separate import statements.

#### `Merge`  
Merge imports sharing the same prefix into a single grouped import
statement.

> You may want to check the [`AggressiveMerge`](OrganizeImports.md#aggressivemerge)
> option for more concise results despite a relatively low risk of introducing
> compilation errors.

> `OrganizeImports` does not support cases where one name is renamed to
> multiple aliases within the same source file when `groupedImports` is
> set to `Merge`. (The IntelliJ IDEA Scala import optimizer does not
> support this either.)
> 
> Scala allows a name to be renamed to multiple aliases within a single
> source file, which makes merging import statements tricky. For example:
>
> ```scala
> import java.lang.{Double => JDouble}
> import java.lang.{Double => JavaDouble}
> import java.lang.Integer
> ```
> 
> The above three imports can be merged into:
> 
> ```scala
> import java.lang.{Double => JDouble}
> import java.lang.{Double => JavaDouble, Integer}
> ```
> 
> but not:
> 
> ```scala
> import java.lang.{Double => JDouble, Double => JavaDouble, Integer}
> ```
> 
> because Scala disallow a name (in this case, `Double`) to appear in one
> import multiple times.
> 
> Here’s a more complicated example:
> 
> ```scala
> import p.{A => A1}
> import p.{A => A2}
> import p.{A => A3}
> 
> import p.{B => B1}
> import p.{B => B2}
> 
> import p.{C => C1}
> import p.{C => C2}
> import p.{C => C3}
> import p.{C => C4}
> ```
> 
> While merging these imports, we may want to "bin-pack" them to minimize
> the number of the result import statements:
> 
> ```scala
> import p.{A => A1, B => B1, C => C1}
> import p.{A => A2, B => B2, C => C2}
> import p.{A => A3, C3 => C3}
> import p.{C => C4}
> ```
> 
> However, in reality, renaming aliasing a name multiple times in the same
> source file is rarely a practical need. Therefore, `OrganizeImports`
> does not support this when `groupedImports` is set to `Merge` to avoid
> the extra complexity.

#### `AggressiveMerge`  
Similar to `Merge`, but merges imports more aggressively and produces
more concise results, despite a relatively low risk of introducing
compilation errors.

The `OrganizeImports` rule tries hard to guarantee correctness in all
cases. This forces it to be more conservative when merging imports, and
may sometimes produce suboptimal output. Here is a concrete example
about correctness:

```scala
import scala.collection.immutable._
import scala.collection.mutable.Map
import scala.collection.mutable._

object Example {
  val m: Map[Int, Int] = ???
}
```

At a first glance, it seems feasible to simply drop the second import
since `mutable._` already covers `mutble.Map`. However, similar to the
example illustrated in the section about the
[`coalesceToWildcardImportThreshold`
option](OrganizeImports.md#coalescetowildcardimportthreshold), the type of `Example.m`
above is `mutable.Map`, because the mutable `Map` explicitly imported in
the second import takes higher precedence than the immutable `Map`
imported via wildcard in the first import. If we merge the last two
imports naively, we’ll get:

```scala
import scala.collection.immutable._
import scala.collection.mutable._
```

This triggers in a compilation error, because both `immutable.Map` and
`mutable.Map` are now imported via wildcards with the same precedence.
This makes the type of `Example.m` ambiguous. The correct result should
be:

```scala
import scala.collection.immutable._
import scala.collection.mutable.{Map, _}
```

On the other hand, the case discussed above is rarely seen in practice.
A more commonly seen case is something like:

```scala
import scala.collection.mutable.Map
import scala.collection.mutable._
```

Instead of being conservative and produce a suboptimal output like:

```scala
import scala.collection.mutable.{Map, _}
```

setting `groupedImports` to `AggressiveMerge` produces
```scala
import scala.collection.mutable._
```

#### `Keep`  
Leave grouped imports and imports sharing the same prefix untouched.

### Default value

`Explode`

Rationale: despite making the import section lengthier, exploding grouped
imports into separate import statements is made the default behavior because
it is more friendly to version control and less likely to create annoying
merge conflicts caused by trivial import changes.

### Examples

#### `Explode`  

```conf
OrganizeImports.groupedImports = Explode
```

Before:

```scala
import scala.collection.mutable.{ArrayBuffer, Buffer, StringBuilder}
```

After:

```scala
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.Buffer
import scala.collection.mutable.StringBuilder
```

#### `Merge`  

```conf
OrganizeImports.groupedImports = Merge
```

Before:

```scala
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.Buffer
import scala.collection.mutable.StringBuilder
import scala.collection.immutable.Set
import scala.collection.immutable._
```

After:

```scala
import scala.collection.mutable.{ArrayBuffer, Buffer, StringBuilder}
import scala.collection.immutable.{Set, _}
```

#### `AggressiveMerge`  

```conf
OrganizeImports.groupedImports = AggressiveMerge
```

Before:

```scala
import scala.collection.mutable.ArrayBuffer
import scala.collection.mutable.Buffer
import scala.collection.mutable.StringBuilder
import scala.collection.immutable.Set
import scala.collection.immutable._
```

After:

```scala
import scala.collection.mutable.{ArrayBuffer, Buffer, StringBuilder}
import scala.collection.immutable._
```

`groups`
--------

Defines import groups by prefix patterns. Only global imports are
processed.

All the imports matching the same prefix pattern are gathered into the
same group and sorted by the order defined by the
[`importsOrder`](OrganizeImports.md#importsorder) option.

> Comments living *between* imports being processed will be *removed*.

> `OrganizeImports` tries to match the longest prefix while grouping
> imports. For instance, the following configuration groups `scala.meta.`
> and `scala.` imports into different two groups properly:
> 
> ```conf
> OrganizeImports.groups = [
>   "re:javax?\\."
>   "scala."
>   "scala.meta."
>   "*"
> ]
> ```

> No matter how the `groups` option is configured, a special
> order-preserving import group may appear after all the configured import
> groups when:
> 
> 1.  The `expandRelative` option is set to `false` and there are relative
>     imports.
> 
> 2.  The `groupExplicitlyImportedImplicitsSeparately` option is set to
>     `true` and there are implicit names explicitly imported.
> 
> This special import group is necessary because the above two kinds of
> imports are order sensitive:
> 
> #### Relative imports  
> For instance, sorting the following imports in alphabetical order
> introduces compilation errors:
> 
> ```scala
> import scala.util
> import util.control
> import control.NonFatal
> ```
> #### Explicitly imported implicit names  
> Please refer to the
> [`groupExplicitlyImportedImplicitsSeparately`](OrganizeImports.md#groupexplicitlyimportedimplicitsseparately)
> option for more details.

### Value type

An ordered list of import prefix pattern strings. A prefix pattern can
be one of the following:

#### A plain-text pattern  
For instance, `"scala."` is a plain-text pattern that matches imports
referring the `scala` package. Please note that the trailing dot is
necessary, otherwise you may have `scalafix` and `scala` imports in the
same group, which is not what you want in most cases.

#### A regular expression pattern  
A regular expression pattern starts with `re:`. For instance,
`"re:javax?\\."` is such a pattern that matches both the `java` and the
`javax` packages. Please refer to the
[`java.util.regex.Pattern`](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html)
Javadoc page for the regular expression syntax. Note that special
characters like backslashes must be escaped.

#### The wildcard pattern  
The wildcard pattern, `"*"`, defines the wildcard group, which matches
all fully-qualified imports not belonging to any other groups. It can be
omitted when it’s the last group. So the following two configurations
are equivalent:

```conf
OrganizeImports.groups = ["re:javax?\\.", "scala.", "*"]
OrganizeImports.groups = ["re:javax?\\.", "scala."]
```

#### A blank line marker  
A blank line marker, `"---"`, defines a blank line between two adjacent
import groups when [`blankLines`](OrganizeImports.md#blanklines) is
set to `Manual`. It is ignored when `blankLines` is `Auto`. Leading and
trailing blank line markers are always ignored. Multiple consecutive blank
line markers are treated as a single one. So the following three configurations
are all equivalent:

```conf
OrganizeImports {
  blankLines = Manual
  groups = [
    "---"
    "re:javax?\\."
    "---"
    "scala."
    "---"
    "---"
    "*"
    "---"
  ]
}

OrganizeImports {
  blankLines = Manual
  groups = [
    "re:javax?\\."
    "---"
    "scala."
    "---"
    "*"
  ]
}

OrganizeImports {
  blankLines = Auto
  groups = [
    "re:javax?\\."
    "scala."
    "*"
  ]
}
```

### Default value

```conf
[
  "*"
  "re:(javax?|scala)\\."
]
```

Rationale: this aligns with the default configuration of the IntelliJ Scala plugin
version 2020.3.

### Examples

#### Fully-qualified imports only  

```conf
OrganizeImports.groups = ["re:javax?\\.", "scala.", "*"]
```

Before:

```scala
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext

import sun.misc.BASE64Encoder
```

#### With relative imports  

```conf
OrganizeImports.groups = ["re:javax?\\.", "scala.", "*"]
```

Before:

```scala
import scala.utilRationale
import util.control
import control.NonFatal
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:
```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext
import scala.util

import sun.misc.BASE64Encoder

import util.control
import control.NonFatal
```

#### With relative imports and an explicitly imported implicit name  

```conf
OrganizeImports {
  groups = ["re:javax?\\.", "scala.", "*"]
  groupExplicitlyImportedImplicitsSeparately = true
}
```

Before:
```scala
import scala.util
import util.control
import control.NonFatal
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext.Implicits.global
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.util

import sun.misc.BASE64Encoder

import util.control
import control.NonFatal
import scala.concurrent.ExecutionContext.Implicits.global
```

#### Regular expression  
Defining import groups using regular expressions can be quite flexible.
For instance, the `scala.meta` package is not part of the Scala standard
library, but the default groups defined in the `OrganizeImports.groups`
option move imports from this package into the `scala.` group. The
following example illustrates how to move them into the wildcard group
using regular expression.

```conf
OrganizeImports.groups = [
  "re:javax?\\."
  "re:scala.(?!meta\\.)"
  "*"
]
```

Before:

```scala
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import scala.meta.Tree
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
import scala.meta.Import
import scala.meta.Pkg
```

After:

```scala
import java.time.Clock
import javax.annotation.Generated

import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext

import scala.meta.Import
import scala.meta.Pkg
import scala.meta.Tree
import sun.misc.BASE64Encoder
```

#### With manually configured blank lines  

```conf
OrganizeImports {
  blankLines = Manual
  groups = [
    "*"
    "---"
    "re:javax?\\."
    "scala."
  ]
}
```

Before:

```scala
import scala.collection.JavaConverters._
import java.time.Clock
import sun.misc.BASE64Encoder
import javax.annotation.Generated
import scala.concurrent.ExecutionContext
```

After:

```scala
import sun.misc.BASE64Encoder

import java.time.Clock
import javax.annotation.Generated
import scala.collection.JavaConverters._
import scala.concurrent.ExecutionContext
```

`importSelectorsOrder`
----------------------

Specifies the order of grouped import selectors within a single import
expression.

### Value type

Enum: `Ascii | SymbolsFirst | Keep`

#### `Ascii`  
Sort import selectors by ASCII codes, equivalent to the
[`AsciiSortImports`](https://scalameta.org/scalafmt/docs/configuration.html#asciisortimports)
rewriting rule in Scalafmt.

#### `SymbolsFirst`  
Sort import selectors by the groups: symbols, lower-case, upper-case,
equivalent to the
[`SortImports`](https://scalameta.org/scalafmt/docs/configuration.html#sortimports)
rewriting rule in Scalafmt.

#### `Keep`  
Keep the original order.

### Default value

`Ascii`

### Examples

#### `Ascii`  

```conf
OrganizeImports {
  groupedImports = Keep
  importSelectorsOrder = Ascii
}
```

Before:

```scala
import foo.{~>, `symbol`, bar, Random}
```

After:

```scala
import foo.{Random, `symbol`, bar, ~>}
```

#### `SymbolsFirst`  

```conf
OrganizeImports {
  groupedImports = Keep
  importSelectorsOrder = SymbolsFirst
}
```

Before:

```scala
import foo.{Random, `symbol`, bar, ~>}
```

After:
```scala
import foo.{~>, `symbol`, bar, Random}
```

`importsOrder`
--------------

Specifies the order of import statements within import groups defined by
the [`OrganizeImports.groups`](#groups) option.

### Value type

Enum: `Ascii | SymbolsFirst | Keep`

#### `Ascii`  
Sort import statements by ASCII codes. This is the default sorting order
that the IntelliJ IDEA Scala import optimizer picks ("lexicographically"
option).

#### `SymbolsFirst`  
Put wildcard imports and grouped imports with braces first, otherwise
same as `Ascii`. This replicates IntelliJ IDEA Scala’s "scalastyle
consistent" option.

#### `Keep`  
Keep the original order.

### Default value

`Ascii`

### Examples

#### `Ascii`  

```conf
OrganizeImports {
  groupedImports = Keep
  importsOrder = Ascii
}
```

Before:

```scala
import scala.concurrent._
import scala.concurrent.{Future, Promise}
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent.duration
```

After:

```scala
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent._
import scala.concurrent.duration
import scala.concurrent.{Promise, Future}
```

#### `SymbolsFirst`  

```conf
OrganizeImports {
  groupedImports = Keep
  importsOrder = SymbolsFirst
}
```

Before:

```scala
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent._
import scala.concurrent.duration
import scala.concurrent.{Promise, Future}
```

After:

```scala
import scala.concurrent._
import scala.concurrent.{Future, Promise}
import scala.concurrent.ExecutionContext.Implicits._
import scala.concurrent.duration
```

`preset`
--------

Specify a preset style.

### Value type

Enum: `DEFAULT | INTELLIJ_2020_3`

#### `DEFAULT`  
An opinionated style recommended for new projects. The `OrganizeImports`
rule tries its best to ensure correctness in all cases when possible.
This default style aligns with this principal. In addition, by setting
`groupedImports` to `Explode`, this style is also more friendly to
version control and less likely to create annoying merge conflicts
caused by trivial import changes.

```conf
OrganizeImports {
  blankLines = Auto
  coalesceToWildcardImportThreshold = null
  expandRelative = false
  groupExplicitlyImportedImplicitsSeparately = false
  groupedImports = Explode
  groups = [
    "*"
    "re:(javax?|scala)\\."
  ]
  importSelectorsOrder = Ascii
  importsOrder = Ascii
  preset = DEFAULT
  removeUnused = true
}
```

#### `INTELLIJ_2020_3`  
A style that is compatible with the default configuration of the
IntelliJ Scala 2020.3 import optimizer. It is mostly useful for adding
`OrganizeImports` to existing projects developed using the IntelliJ
Scala plugin. However, the configuration of this style may introduce
subtle correctness issues (so does the default configuration of the
IntelliJ Scala plugin). Please see the
[`coalesceToWildcardImportThreshold`
option](OrganizeImports.md#coalescetowildcardimportthreshold)
for more details.

```conf
OrganizeImports {
  blankLines = Auto
  coalesceToWildcardImportThreshold = 5
  expandRelative = false
  groupExplicitlyImportedImplicitsSeparately = false
  groupedImports = Merge
  groups = [
    "*"
    "re:(javax?|scala)\\."
  ]
  importSelectorsOrder = Ascii
  importsOrder = Ascii
  preset = INTELLIJ_2020_3
  removeUnused = true
}
```

### Default value

`DEFAULT`

`removeUnused`
--------------

Remove unused imports.

> The `removeUnused` option doesn’t play perfectly with the `expandRelative`
> option. Setting `expandRelative` to `true` might introduce new unused
> imports (see [`expandRelative`](OrganizeImports.md#expandrelative)).
> These newly introduced unused imports cannot be removed by setting
> `removeUnused` to `true`. This is because unused imports are identified
> using Scala compilation diagnostics information, and the compilation phase
> happens before Scalafix rules get applied.

> The `removeUnused` option is currently not supported for source files
> compiled with Scala 3, as the [compiler cannot issue warnings for unused
> imports
> yet](https://docs.scala-lang.org/scala3/guides/migration/options-lookup.html#warning-settings).
> As a result, you must set `removeUnused` to `false` when running the
> rule on source files compiled with Scala 3.

### Value type

Boolean

### Default value

`true`

### Examples

```conf
OrganizeImports {
  groups = ["javax?\\.", "scala.", "*"]
  removeUnused = true // not supported in Scala 3
}
```

Before:

```scala
import scala.collection.mutable.{Buffer, ArrayBuffer}
import java.time.Clock
import java.lang.{Long => JLong, Double => JDouble}

object RemoveUnused {
  val buffer: ArrayBuffer[Int] = ArrayBuffer.empty[Int]
  val long: JLong = JLong.parseLong("0")
}
```

After:

```scala
import java.lang.{Long => JLong}

import scala.collection.mutable.ArrayBuffer

object RemoveUnused {
  val buffer: ArrayBuffer[Int] = ArrayBuffer.empty[Int]
  val long: JLong = JLong.parseLong("0")
}
```
