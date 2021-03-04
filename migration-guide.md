# Cats Effect 3 Migration Guide

<!--
some resources to help with writing this

the issue
https://github.com/typelevel/cats-effect/issues/1048

the proposal by Daniel https://github.com/typelevel/cats-effect/issues/634

https://scalacenter.github.io/scala-3-migration-guide/
https://ionicframework.com/docs/reference/migration
https://v3.vuejs.org/guide/migration/introduction.html#overview
https://medium.com/storybookjs/storybook-6-migration-guide-200346241bb5

start with modules: most people want core. library author / application developer tracks?

 -->

 <!-- random note: there should be a clear sequence of steps for those who don't really want to read through the overview and whatnot -->
 <!-- something like "letsdodissssss": update deps to compatible versions, run scalafix, use lookup table to update any outstanding method calls... -->

### Before you begin: this isn't a "quick start" guide

This guide is meant for existing users of Cats Effect 2 who want to upgrade their applications
to the 3.x series of releases, starting with 3.0.0.

> If you haven't used Cats Effect before
> and want to give it a try, please follow the [quick start guide](dead-link) instead!

In this guide, we will not discuss the new features and additions in the library,
but focus on changes the users will need to make to get their projects to build with Cats Effect 3.
For new features, please consult the [documentation](dead-link) instead.

## Overview

Cats Effect 3 (CE3 for short) is a complete redesign of the library.
Some abstractions known from Cats Effect 2 (CE2) have been removed, others changed responsibilities, and finally, new abstractions were introduced.

The `cats.effect.IO` type known from CE2 is still there, albeit with a different place in the type class hierarchy - namely, it doesn't appear in it.
The new set of type classes has been designed in a way that deliberately avoids coupling with `IO` in any shape or form, which has several advantages.

<!-- todo should we mention them here? What I had in mind was a more parametric hierarchy (which leads to nicer laws), smaller dependency footprint (e.g. ZIO users could now have a single IO in the classpath even with interop) -->

### Modules

Cats Effect 2 had a single module for non-test code:

```scala
"org.typelevel" %% "cats-effect" % "2.3.1",
// For law testing
"org.typelevel" %% "cats-effect-laws" % "2.3.1" % Test
```

Cats Effect 3 splits that into multiple modules:

```scala
"org.typelevel" %% "cats-effect-kernel"         % "3.0.0-RC1",
"org.typelevel" %% "cats-effect-kernel-testkit" % "3.0.0-RC1" % Test,
"org.typelevel" %% "cats-effect-laws"           % "3.0.0-RC1" % Test,
"org.typelevel" %% "cats-effect"                % "3.0.0-RC1",
"org.typelevel" %% "cats-effect-testkit"        % "3.0.0-RC1" % Test,
"org.typelevel" %% "cats-effect-std"            % "3.0.0-RC1"
```

Depending on how you use Cats Effect, you might be able to pick just some of them.

#### I am a library author, only using the type classes

Use `kernel`. For your tests, you might also need `kernel-testkit`, and `core` (for `IO`) or another effect library.

#### I am a library author, implementing the type classes

Use `kernel` and `laws`.

#### I am an application developer or library author, using IO

Use `core`. If you were using `Semaphore` previously, you will also need `std`.

## Scalafix migration

Many parts of this migration can be automated by using the Scalafix migration.

> Note: In case of projects using Scala Steward, the migration should automatically be applied
when you receive the update.

If you want to trigger the migration manually:

todo. WIP in [Frank's PR](https://github.com/typelevel/cats-effect/pull/1686)
<!-- todo instructions -->

## Notable changes

Here's the new type class hierarchy. It might be helpful in understanding some of the changes:

<!-- todo make it just a link? -->
[![hierarchy](https://raw.githubusercontent.com/typelevel/cats-effect/series/3.x/images/hierarchy.svg)](https://raw.githubusercontent.com/typelevel/cats-effect/series/3.x/images/hierarchy.svg)

Most of the following are handled by [the Scalafix migration](dead-link). If you can, try that first!

Note: package name changes were skipped from the table. Most type classes are now in `cats.effect.kernel`.

| Cats Effect 2.x                             | Cats Effect 3                           | Notes                                                                  |
| ------------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `Async[F].async`                            | `Async[F].async_`                       |
| `Async[F].asyncF(f)`                        | `Async[F].async(f).as(none)`            |
| `Async.shift`                               | -                                       | See [below](#shifting)                                                 |
| `Async.fromFuture`                          | `Async[F].fromFuture`                   |
| `Async.memoize`                             | `Concurrent[F].memoize`                 |
| `Async.parTraverseN`                        | `Concurrent[F].parTraverseN`            |
| `Async.parSequenceN`                        | `Concurrent[F].parSequenceN`            |
| `Async[F].liftIO`, `Async.liftIO`           | `LiftIO[F].liftIO`                      | `LiftIO` is in the `cats-effect` module                                |
| `Async <: LiftIO`                           | No subtyping relationship               | `LiftIO` is in the `cats-effect` module                                |
| `Blocker.apply`                             | -                                       | blocking pool is provided by runtime                                   |
| `Blocker.delay`                             | `Sync[F].blocking`                      | `Blocker` was removed                                                  |
| `Blocker(ec).blockOn(fa)`                   | `Async[F].evalOn(fa, ec)`               | You can probably use `Sync[F].blocking`                                |
| `Blocker.blockOnK`                          | -                                       | <!-- todo we should have this in Async -->                             |
| `Bracket[F].bracket`                        | `MonadCancel[F].bracket`                |
| `Bracket[F].bracketCase`                    | `MonadCancel[F].bracketCase`            | `ExitCase` is now `Outcome`                                            |
| `Bracket[F].uncancelable(fa)`               | `MonadCancel[F].uncancelable(_ => fa)`  |
| `Bracket[F].guarantee`                      | `MonadCancel[F].guarantee`              |
| `Bracket[F].guaranteeCase`                  | `MonadCancel[F].guaranteeCase`          | `ExitCase` is now `Outcome`                                            |
| `Bracket[F].onCancel`                       | `MonadCancel[F].onCancel`               |
| `CancelToken[F]`                            | `F[Unit]`                               |
| `Clock[F].realTime: TimeUnit => F[Long]`    | `Clock[F].realTime: F[FiniteDuration]`  |
| `Clock[F].monotonic: TimeUnit => F[Long]`   | `Clock[F].monotonic: F[FiniteDuration]` |
| `Clock.instantNow`                          | `Clock[F].realTimeInstant`              |
| `Clock.create`, `Clock[F].mapK`             | -                                       | See [below](#clock-changes)                                            |
| `Concurrent[F].start`                       | `Spawn[F].start`                        |
| `Concurrent[F].background`                  | `Spawn[F].background`                   | Value in resource is now an `Outcome`                                  |
| `Concurrent[F].liftIO`, `Concurrent.liftIO` | `LiftIO[F].liftIO`                      | `LiftIO` is in the `cats-effect` module                                |
| `Concurrent <: LiftIO`                      | No subtyping relationship               | `LiftIO` is in the `cats-effect` module                                |
| `Concurrent[F].race`                        | `Spawn[F].race`                         |
| `Concurrent[F].racePair`                    | `Spawn[F].racePair`                     |
| `Concurrent[F].cancelable`                  | `Async.async(f)`                        | Wrap side effects in F, cancel token in `Some`                         |
| `Concurrent[F].cancelableF`                 | `Async.async(f(_).some)`                | `Some`                                                                 |
| `Concurrent[F].continual`                   | see [below](#concurrent:-continual)     | <!-- <-------    todo deadlink  -->                                    |
| `Concurrent.continual`                      | see [below](#concurrent:-continual)     | <!-- <-------    todo deadlink  -->                                    |
| `Concurrent.timeout`                        | `Temporal[F].timeout`                   |
| `Concurrent.timeoutTo`                      | `Temporal[F].timeoutTo`                 |
| `Concurrent.memoize`                        | `Concurrent[F].memoize`                 |
| `Concurrent.parTraverseN`                   | `Concurrent[F].parTraverseN`            |
| `Concurrent.parSequenceN`                   | `Concurrent[F].parSequenceN`            |
| `ConcurrentEffect[F]`                       | `cats.effect.std.Dispatcher`            | See [below](#dispatcher)                                               |
| `ContextShift[F].shift`                     | See [below](#shifting)                  |
| `ContextShift[F].evalOn`                    | `Async[F].evalOn`                       |
| `ContextShift.evalOnK`                      | -                                       | <!-- https://github.com/typelevel/cats-effect/issues/1722 -->          |
| `Effect[F]`                                 | `cats.effect.std.Dispatcher`            | See [below](#dispatcher)                                               |
| `Effect.toIOK`                              | -                                       | See [below](#dispatcher)                                               |
| `ExitCase[E]`                               | `Outcome[F, E, A]`                      | See [below](#outcome)                                                  |
| `Fiber[F, A]`                               | `Fiber[F, E, A]`                        | See [below](#outcome)                                                  |
| `Fiber[F, A].join: F[A]`                    | `Fiber[F, E, A].joinWithNever`          | See [below](#outcome)                                                  |
| `Sync[F].suspend`                           | `Sync[F].defer`                         |
| `SyncEffect`                                | -                                       | See [below](#dispatcher)                                               |
| `IO#as`                                     | `IO.as` / `IO.map`                      | the argument isn't by-name anymore                                     |
| `IO.runAsync`, `IO.runCancelable`           | -                                       | Use unsafe variants or [`Dispatcher`](#dispatcher)                     |
| `IO.unsafe*`                                | The same or `Dispatcher`                | Methods that run an IO require an implicit `IORuntime`                 |
| `IO.unsafeRunAsyncAndForget`                | `IO.unsafeRunAndForget`                 |
| `IO.unsafeRunCancelable`                    | `start.unsafeRunSync.cancel`            |
| `IO.unsafeRunTimed`                         | -                                       |
| `IO.background`                             | The same                                | Value in resource is now an `Outcome`                                  |
| `IO.guaranteeCase`/`bracketCase`            | The same                                | `ExitCase` is now `Outcome`                                            |
| `IO.parProduct`                             | `IO.both`                               |
| `IO.suspend`                                | `IO.defer`                              |
| `IO.none[A]`                                | IO.pure(none[A])`                       | [Might be added](https://github.com/typelevel/cats-effect/issues/1728) |
| `IO.shift`                                  | See [below](#shifting)                  |
| `IO.cancelBoundary`                         | `IO.cede`                               | Also [shifts](#shifting)                                               |
| IO tracing                                  | Currently missing                       |
| `Resource.parZip`                           | `Resource.both`                         |
| `Resource.fromAutoCloseableBlocking`        | `Resource.fromAutoCloseable`            | The method always uses `blocking` for the cleanup action               |
| `Timer[F].clock`                            | `Clock[F]`                              |
| `Timer[F].sleep`                            | `Temporal[F].sleep`                     |

TODO: IO,IOApp, Resource, Timer

However, some changes will require more work than a simple search/replace.
We will go through them here.

### Concurrent: continual

<!-- todo explain -->

```scala
def continual[A, B](fa: F[A])(f: Either[Throwable, A] => F[B]): F[B] = MonadCancel[F].uncancelable { poll =>
  poll(fa).attempt.flatMap(f)
}
```

### Dispatcher

todo - Gavin wrote about this

## Compatibility with Cats Effect 2

### Binary compatibility

There is none! The library was rewritten from scratch, and there was no goal of having binary compatibility with pre-3.0 releases.

> Note: We will guarantee binary compatibility between all stable releases in the 3.x series, and a 2.x branch will be maintained for some time to allow a smoother transition.

What this means for you: if you are an end user (an application developer),
you will need to update **every library using cats-effect** to a CE3-compatible version before you can safely deploy your application.
[We are keeping track of the efforts of library authors to publish compatible releases](https://github.com/typelevel/cats-effect/issues/1330) as soon as possible when 3.0.0 final is out.

If you are a library author, you also should guarantee your dependencies are CE3-compatible before you publish a release.

To get some aid in pinpointing problematic dependencies, <!-- todo this is just for sbt users --> we recommend using existing tooling like
[`sbt`'s eviction mechanism](https://www.scala-sbt.org/1.x/docs/Library-Management.html#Eviction+warning) and
[the dependency graph plugin included in `sbt` since 1.4.0](https://www.scala-sbt.org/1.x/docs/sbt-1.4-Release-Notes.html#sbt-dependency-graph+is+in-sourced). Using the `whatDependsOn` task, you will be able to quickly see the libraries that pull in the problematic version.

You might also want to consider [sbt-missinglink](https://github.com/scalacenter/sbt-missinglink) to verify your classpath works with your code, or
[follow the latest developments in sbt's eviction mechanism](https://github.com/sbt/sbt/pull/6221#issuecomment-777722540).

To sum up, the only thing guaranteed when it comes to binary compatibility between CE2 and CE3 is that your code will blow up in runtime if you try to use them together! Make sure to double-check everything your build depends on is updated.

### Source compatibility

In some areas, the code using CE3 looks similarly or identically as it would before the migration.
Regardless of that, it is not recommended to assume anything is the same unless explicitly mentioned in this guide.

### Feature compatibility

Everything that was possible with CE2 should still be possible with CE3, with the following exceptions:

#### Implementing Async

Types that used to implement `Async` but not `Concurrent` from CE2 might not be able to implement anything more than `Sync` in CE3 -
this has an impact on users who have used e.g.
[doobie](https://github.com/tpolecat/doobie)'s `ConnectionIO`, `slick.dbio.DBIO` with
[slick-effect](https://github.com/kubukoz/slick-effect), or
[ciris](https://cir.is)'s `ConfigValue` in a polymorphic context with an `Async[F]` constraint.

Please refer to each library's appropriate documentation to see how to adjust your code to this change.
<!-- todo We might have direct links to the appropriate migration guides here later on -->

#### shifting

The `IO.shift` / `ContextShift[F].shift` methods are gone, and they don't have a fully compatible counterpart.

In CE2, `shift` would ensure the rest of the fiber would be scheduled on the `ExecutionContext` instance (or the `ContextShift` instance) provided in the parameter. This was used for two reasons:

- to switch back from a thread pool not managed by the effect system (e.g. a callback handler in a Java HTTP client)
- to reschedule the fiber on the given `ExecutionContext`, which would give other fibers a chance to run on that context's threads. This is called yielding to the scheduler.

There is no longer a need for the former (shifting back), because interop with callback-based libraries is done through methods in `Async`, which now **switch back to the appropriate thread pool automatically**.

The latter (yielding back to the scheduler) should now be done with `Spawn[F].cede`.

#### Clock changes

todo
<!-- why `create` and mapK are gone (because it's a typeclass now)  -->
