# Gradle Dependency Lock (Experimental)

## Purpose

This project is an experimental major overhaul of the [gradle-dependency-lock-plugin](https://github.com/nebula-plugins/gradle-dependency-lock-plugin).

## Usage

Clone and build with:

    ./gradlew publishToMavenLocal

To apply this plugin:

    buildscript {
        repositories { mavenLocal() }
        dependencies {
            classpath 'com.netflix.nebula:nebula-lock-experimental:latest.release'
        }

        configurations.classpath.resolutionStrategy.cacheDynamicVersionsFor 0, 'minutes'
    }

    apply plugin: 'nebula.lock-experimental'

## Why the change?

(1) At various times, [gradle-dependency-lock-plugin](https://github.com/nebula-plugins/gradle-dependency-lock-plugin) served functions that are now
best served by other means, e.g. version alignment which is now satisfied by [gradle-resolution-rules](https://github.com/nebula-plugins/gradle-resolution-rules-plugin).

(2) **dependencis.lock** is hard to read and separate from where users define their dependencies in **build.gradle**. Unaware
that nebula.dependency-lock is in effect, engineers are often confused about why they don't get a version they expect when they update
 **build.gradle**.

(3) Manually updating a particular dependency while leaving the rest locked requires a-priori knowledge of
 the existence of `./gradlew updateLock -PdependencyLock.updateDependencies=com.example:foo,com.example:bar` mechanism. Well meaning
 engineers may attempt to update **dependencies.lock** manually, not realizing that the dependency they are attempting to update
 is reflected once per configuration that it participates in, and their manual effort fails to achieve their goal.

(4) nebula.dependency-lock applies its locks using `resolutionStrategy.force`, which can lead to ordering problems with other
plugins that affect `resolutionStrategy`.

## The `lock` extension method

The experimental plugin hangs a method on `org.gradle.api.artifacts.Dependency` called `lock` that takes a single `String` parameter with
the locked version. The `lock` method receives the `Dependency` created and immediately substitutes it with a `Dependency` with the locked version.
This happens before any other dependency-affecting event in the Gradle ecosystem.

In effect, `lock` makes it seem to Gradle that you that rather than writing a dynamic constraint like **latest.release** you had actually just written
in a fixed version. We believe this is the real root use case for dependency locks: I want to write static versions into my
dependency blocks, but I don't want to have to manually update the static versions as new versions come out.

## The `updateLocks` task

Running `./gradlew updateLocks` resolves each configuration without locks, and uses an AST parser to locate first order
dependencies in your **build.gradle** and write out the appropriate `lock` method call with the resolved version. `updateLocks`
also detects if you change the unlocked version to a static constraint and removes the `lock` method call.

Because Gradle returns `null` from a method call like

    compile 'com.google.guava:19.+',
            'commons-lang:commons-lang:2.+'

the task will generate the following:

    compile 'com.google.guava:19.+' lock '19.0'
    compile 'commons-lang:commons-lang:2.+' lock '2.6'

## Ignoring locks

Running with `-PdependencyLock.ignore` causes the lock method to short-circuit and leave dynamic constraints in affect.

## Migration from the legacy plugin

Running `./gradlew convertLegacyLocks` uses an AST parser to locate first order dependencies matching **dependencies.lock**
entries and adds the appropriate `lock` method call to your **build.gradle**.

## What about locking transitives?

We no longer believe there are any reasons to lock transitive dependencies, because

(1) The vast majority of Java libraries are published with fixed versions (and Gradle does not support Ivy's `revConstraint`)
(2) Responsibility for version alignment has been externalized to [gradle-resolution-rules](https://github.com/nebula-plugins/gradle-resolution-rules-plugin).
