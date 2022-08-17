[![Actions Status](https://github.com/lizmat/rak/actions/workflows/test.yml/badge.svg)](https://github.com/lizmat/rak/actions)

NAME
====

rak - plumbing to be able to look for stuff

SYNOPSIS
========

```raku
use rak;

# look for "foo" in all .txt files from current directory
for rak / foo /, :file(/ \.txt $/) -> (:key($path), :value(@found)) {
    if @found {
        say "$path:";
        say .key ~ ':' ~ .value for @found;
    }
}
```

DESCRIPTION
===========

The `rak` subroutine provides a mostly abstract core search (plumbing) functionality to be used by modules such as (porcelain) `App::Rak`.

THEORY OF OPERATION
===================

The `rak` subroutine basically goes through 6 steps to produce a result.

### 1. Acquire sources

The first step is determining the objects that should be searched for the specified pattern. If an object is a `Str`, it will be assume that it is a path specification of a file to be searched in some form and an `IO::Path` object will be created for it.

Related named arguments are (in alphabetical order):

  * :dir - filter for directory basename check to include

  * :file - filter for file basename check to include

  * :files-from - file containing filenames as source

  * :paths - paths to recurse into if directory

  * :paths-from - file containing paths to recurse into

  * :recurse-symlinked-dir - recurse into symlinked directories

  * :recurse-unmatched-dir - recurse into directories not matching :dir

  * :sources - list of objects to be considered as source

The result of this step, is a (potentially lazy and hyperable) sequence of objects.

### 2. Filter applicable objects

Filter down the list of sources from step 1 on any additional filesystem related properties. This assumes that the list of objects created are strings of absolute paths to be checked.

  * :accessed - when was path last accessed

  * :blocks- number of filesystem blocks

  * :created - when was path created

  * :device-number - device number on which path is located

  * :filesize - size of the path in bytes

  * :gid - numeric gid of the path

  * :hard-links - number of hard-links to path on filesystem

  * :inode - inode of path on filesystem

  * :is-empty - is path empty (filesize == 0)

  * :is-executable - is path executable

  * :is-git-repo - is path top directory of a Git repository

  * :is-github-repo - is path top directory of a GitHub repository

  * :is-group-executable - is path executable by group

  * :is-group-readable - is path readable by group

  * :is-group-writable - is path writable by group

  * :is-owned-by-group - is path owned by group of current user

  * :is-owned-by-user - is path owned by current user

  * :is-readable - is path readable by current user

  * :is-symbolic-link - is path a symbolic link

  * :is-world-executable - is path executable by any user

  * :is-world-readable - is path readable by any user

  * :is-world-writable - is path writable by any user

  * :is-writable - is path writable by current user

  * :meta-modified - when meta information of path was modified

  * :mode - the mode of the path

  * :modified - when path was last modified

  * :uid - numeric uid of path

The result of this step, is a (potentially lazy and hyperable) sequence of objects.

### 3. Produce items to search in (apply transformers)

The second step is to create the logic for creating items to search in from the objects in step 2. If search is to be done per object, then `.slurp` is called on the object. Otherwise `.lines` is called on the object. Unless one provides their own logic for producing items to search in.

Related named arguments are (in alphabetical order):

  * :encoding - encoding to be used when creating items

  * :find - map sequence of step 1 to item producer

  * :produce-one - produce one item per given source

  * :produce-many - produce zero or more items by given source

  * :omit-item-numbers - do not store item numbers in result

  * :with-line-ending - produce lines with line endings

The result of this step, is a (potentially lazy and hyperable) sequence of objects.

### 4. Create logic for matching

Take the logic of the pattern `Callable`, and create a `Callable` to do the actual matching with the items produced in step 3.

Related named arguments are (in alphabetical order):

  * :invert-match - invert the logic of matching

  * :quietly - absorb any warnings produced by the matcher

  * :silently - absorb any output done by the matcher

  * :stats - produce results and statistics

  * :stats-only - don't produce results, just statistics

### 5. Create logic for running

Take the matcher logic of the `Callable` of step 4 and create a runner `Callable` that will produce the items found and their possible context (such as extra items before or after). Assuming no context, the runner changes a return value of `False` from the matcher into `Empty`, a return value of `True` in the original item, and passes through any other value.

Related named arguments are (in alphabetical order):

  * :after-context - number of items to show after a match

  * :before-context - number of items to show before a match

  * :context - number of items to show around a match

  * :paragraph-context - items around match until false item

  * :passthru-context - pass on *all* items

  * :max-matches-per-source - max # of matches per source

Matching items are represented by `PairMatched` objects, and items that have been added because of the above context arguments, are represented by `PairContext` objects.

### 6. Run the sequence(s)

The final step is to take the `Callable` of step 5 and run that repeatedly on the sequence of step 2, and for each item of that sequence, run the sequence of step 5 on that. Make sure any phasers (`FIRST`, `NEXT` and `LAST`) are called at the appropriate time in a thread-safe manner.

Either produces a sequence in which the key is the source, and the value is a `Slip` of `Pair`s where the key is the item-number and the value is item with the match, or whatever the pattern matcher returned.

Or, produces sequence of whatever a specified mapper returned and/or with uniqueifying enabled.

Related named arguments are (in alphabetical order):

  * :mapper - code to map results of a single source

  * :map-all - also call mapper if a source has no matches

  * :unique - only return unique matches

EXPORTED SUBROUTINES
====================

rak
---

The `rak` subroutine takes a `Callable` (or `Regex`) pattern as the only positional argument and quite a number of named arguments. Or it takes a `Callable` (or `Regex`) as the first positional argument for the pattern, and a hash with named arguments as the second positional argument. In the latter case, the hash will have the arguments removed that the `rak` subroutine needed for its configuration and execution.

### Return value

A `Pair` is always returned: if the key is an `Exception` then that means that somehow that Exception was thrown during processing, In that case the value contains the exception's message.

Or the key is a `Map`, in which case that `Map` contains the statistics of the search (if activated with `:stats` or `:stats-only`, or else an empty `Map`), and the value contains `Nil` (if :stats-only was specified) or the result `Iterable` of the search.

#### Result Iterable

If the `:unique` argument was specified with a trueish value, then the result `Iterable` will just produce the unique values that were either explicitely returned by the matcher, or the unique items that matched.

Otherwise the value is an `Iterable` of `Pair`s which contain the source object as key (by default the absolute path of the file in which the pattern was found).

If there was only one item produced per source, then the value will be that value, or whatever was produced by the matching logic.

Otherwise the value will be a `Slip`. If no item numbers were requested, each element contains the result of matching. If item-numbers were requested, each element is a `Pair`, of which the key is the item-number where the pattern was found, and the value is the product of the search (which, by default, is the item in which the pattern was found).

Possible result structures in a graph:

    rak
     |- Pair
         |- key: Exception object
         |- value: exception message
         or
         |- key: - empty Map
         |      or
         |       - Map with statistics
         |
         |- value: - Nil
                  or
                   - result Iterable
                      |- Pair
                      |   |- key: source object
                      |   |- value:
                      |        |- Slip
                      |        |   |- PairMatched|PairContext
                      |        |   |   |- key: item-number
                      |        |   |   \- value: match result
                      |        |   or
                      |        |   \- match result
                      |        or
                      |        \- single item match result
                      or
                      \- unique match result

### Named Arguments

The following named arguments can be specified (in alphabetical order):

#### :accessed(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **access** time of the path. The `Callable` is passed a `Num` value of the access time (number of seconds since epoch) and is expected to return a trueish value to have the path be considered for further selection.

#### :after-context(N)

Indicate the number of items that should also be returned **after** an item with a pattern match. Defaults to **0**.

#### :batch(N)

When hypering over multiple cores, indicate how many items should be processed per thread at a time. Defaults to whatever the system thinks is best (which **may** be sub-optimal).

#### :before-context(N)

Indicate the number of items that should also be returned **before** an item with a pattern match. Defaults to **0**.

#### :blocks(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **number of blocks** used by the path on the filesystem on which the path is located. The `Callable` is passed the number of blocks of a path and is expected to return a trueish value to have the path be considered for further selection.

#### :context(N)

Indicate the number of items that should also be returned around an item with a pattern match. Defaults to **0**.

#### :created(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **creation** time of the path. The `Callable` is passed a `Num` value of the creation time (number of seconds since epoch) and is expected to return a trueish value to have the path be considered for further selection.

#### :degree(N)

When hypering over multiple cores, indicate the maximum number of threads that should be used. Defaults to whatever the system thinks is best (which **may** be sub-optimal).

#### :device-number(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **device number** of the path. The `Callable` is passed the device number of the device on which the path is located and is expected to return a trueish value to have the path be considered for further selection.

#### :dir(&dir-matcher)

If specified, indicates the matcher that should be used to select acceptable directories with the `paths` utility. Defaults to `True` indicating **all** directories should be recursed into. Applicable for any situation where `paths` is used to create the list of files to check.

#### dont-catch

Flag. If specified with a trueish value, will **not** catch any error during processing, but will throw any error again. Defaults to `False`, making sure that errors **will** be caught.

#### :encoding("utf8-c8")

When specified with a string, indicates the name of the encoding to be used to produce items to check (typically by calling `lines` or `slurp`). Defaults to `utf8-c8`, the UTF-8 encoding that is permissive of encoding issues.

#### :file(&file-matcher)

If specified, indicates the matcher that should be used to select acceptable files with the `paths` utility. Defaults to `True` indicating **all** files should be checked. Applicable for any situation where `paths` is used to create the list of files to check.

#### :filesize(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **number of bytes** of the path. The `Callable` is passed the number of bytes of a path and is expected to return a trueish value to have the path be considered for further selection.

#### :files-from($filename)

If specified, indicates the name of the file from which a list of files to be used as sources will be read.

#### :find

Flag. If specified, maps the sources of items into items to search.

#### :gid(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **gid** of the path. The `Callable` is passed the numeric gid of a path and is expected to return a trueish value to have the path be considered for further selection. See also `owner` and `group` filters.

#### :hard-links(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **number of hard-links** of the path. The `Callable` is passed the number of hard-links of a path and is expected to return a trueish value to have the path be considered for further selection.

#### :inode(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **inode** of the path. The `Callable` is passed the inode of a path and is expected to return a trueish value to have the path be considered for further selection.

#### :invert-match

Flag. If specified with a trueish value, will negate the return value of the pattern if a `Bool` was returned. Defaults to `False`.

#### :is-empty

Flag. If specified, indicates paths, that are empty (aka: have a filesize of 0 bytes), are (not) acceptable for further selection. Usually only makes sense when uses together with `:find`.

#### :is-executable

Flag. If specified, indicates paths, that are **executable** by the current **user**, are (not) acceptable for further selection.

#### :is-git-repo

Flag. If specified, indicates paths, look like they're the top directory in a Git repository (because they have a `.git` directory in it), are (not) acceptable for further selection.

#### :is-github-repo

Flag. If specified, indicates paths, look like they're the top directory in a GitHub repository (because they have a `.github` directory in it), are (not) acceptable for further selection.

#### :is-group-executable

Flag. If specified, indicates paths, that are **executable** by the current **group**, are (not) acceptable for further selection.

#### :is-group-readable

Flag. If specified, indicates paths, that are **readable** by the current **group**, are (not) acceptable for further selection.

#### :is-group-writable

Flag. If specified, indicates paths, that are **writable** by the current **group**, are (not) acceptable for further selection.

#### :is-readable

Flag. If specified, indicates paths, that are **readable** by the current **user**, are (not) acceptable for further selection.

#### :is-owned-by-group

Flag. If specified, indicates only paths that are **owned** by the **group** of the current user, are (not) acceptable for further selection.

#### :is-owned-by-user

Flag. If specified, indicates only paths that are **owned** by the current **user**, are (not) acceptable for further selection.

#### :is-symbolic-link

Flag. If specified, indicates only paths that are symbolic links, are (not) acceptable for further selection.

#### :is-world-executable

Flag. If specified, indicates paths, that are **executable** by any user or group, are (not) acceptable for further selection.

#### :is-world-readable

Flag. If specified, indicates paths, that are **readable** by any user or group, are (not) acceptable for further selection.

#### :is-world-writeable

Flag. If specified, indicates paths, that are **writable** by any user or group, are (not) acceptable for further selection.

#### :is-writable

Flag. If specified, indicates paths, that are **writable** by the current **user**, are (not) acceptable for further selection.

#### :mapper(&mapper)

If specified, indicates the `Callable` that will be called (in a thread-safe manner) for each source, with the matches of that source. The `Callable` is passed the source object, and a list of matches, if there were any matches. If you want the `Callable` to be called for every source, then you must also specify `:map-all`.

Whatever the mapper `Callable` returns, will become the result of the call to the `rak` subroutine. If you don't want any result to be returned, you can return `Empty` from the mapper `Callable`.

#### :map-all

Flag. If specified with a trueish value, will call the mapper logic, as specified with `:mapper`, even if a source has no matches. Defaults to `False`.

#### :max-matches-per-source(N)

Indicate the maximum number of items that may be produce per source. Defaults to **all** (which can also be specified by an falsish value).

#### :meta-modified(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **modification** time of the path. The `Callable` is passed a `Num` value of the modification time (number of seconds since epoch) and is expected to return a trueish value to have the path be considered for further selection.

#### :mode(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **mode** of the path. The `Callable` is passed the mode of a path and is expected to return a trueish value to have the path be considered for further selection. This is really for advanced types of tests: it's probably easier to use any of the `readable`, `writeable` and `executable` filters.

#### :modified(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **modification** time of the path. The `Callable` is passed a `Num` value of the modification time (number of seconds since epoch) and is expected to return a trueish value to have the path be considered for further selection.

#### :omit-item-numbers

Flag. If specified with a trueish value, won't produce any `PairMatched` or `PairContext` objects in the result, but will just produce the result of the match. Defaults to `False`, meaning to include item numbers.

#### :paragraph-context

Flag. If specified with a trueish value, produce items **around** the empty with a pattern match until a falsish item is encountered.

#### :passthru-context

Flag. If specified with a trueish value, produces **all** items.

#### :paths-from($filename)

If specified, indicates the name of the file from which a list of paths to be used as the base of the production of filename with a `paths` search.

#### :paths(@paths)

If specified, indicates a list of paths that should be used as the base of the production of filename with a `paths` search. If there is no other sources specification (from either the `:files-from`, `paths-from` or `sources`) then the current directory (aka ".") will be assumed.

If a single hyphen is specified as the path, then STDIN will be assumed as the source.

#### :produce-many(&producer)

If specified, indicates a `Callable` that will be called given a source, and is expected to produce zero or more items to be inspected. Defaults to a producer that calles the `lines` method on a given source, with the `:encoding` and `:with-line-ending` arguments.

#### :produce-one(&producer) If specified, indicates a `Callable` that will be called given a source, and is expected to produce one items to be inspected.

#### :recurse-symlinked-dir

Flag. If specified with a trueish value, will recurse into directories that are actually symbolic links. The default is `False`: do **not** recurse into symlinked directories.

#### :recurse-unmatched-dir

Flag. If specified with a trueish value, will recurse into directories that did **not** pass the :<dir>. No files will ever be produced from such directories, but further recursion will be done if directories are encountered. The default is `False`: do **not** recurse into directories that didn't match the `:dir` specification.

#### :quietly

Flag. If specified with a trueish value, will absorb any warnings that may occur when looking for the pattern.

#### :silently("out,err")

When specified with `True`, will absorb any output on STDOUT and STDERR. Optionally can only absorb STDOUT ("out"), STDERR ("err") and both STDOUT and STDERR ("out,err").

#### :sources(@objects)

If specified, indicates a list of objects that should be used as a source for the production of items. Which generally means they cannot be just strings.

#### :stats

Flag. If specified with a trueish value, will keep stats on number of files and number of items seen. And instead of just returning the results sequence, will then return a `List` of the result sequence as the first argument, and a `Map` with statistics as the second argument, with the following keys:

  * nr-sources - number of sources seen

  * nr-items - number of items inspected

  * nr-matches - number of items that matched

  * nr-passthrus - number of items that have passed through

  * nr-changes - number of items that would have been changed

#### :stats-only

Flag. If specified with a trueish value, will perform all searching, but only update counters and not produce any results other than a `Map` with the same keys as with the `:stats-only` flag.

#### :uid(&filter)

If specified, indicates the `Callable` filter that should be used to select acceptable paths by the **uid** of the path. The `Callable` is passed the numeric uid of a path and is expected to return a trueish value to have the path be considered for further selection. See also `owner` and `group` filters.

#### :unique

Flag. If specified, indicates that only unique matches will be returned, instead of the normal sequence of source => result pairs.

#### :with-line-ending

Flag. If specified, indicates line endings are to be kept when producing items to check. Defaults to `False`, meaning that line endings are removed from items to check. Only applicable with line-based checking.

PATTERN RETURN VALUES
---------------------

The return value of the pattern `Callable` match is interpreted in the following ways:

### True

If the `Bool`ean True value is returned, assume the pattern is found. Produce the item unless `:invert-match` was specified.

### False

If the `Bool`ean False value is returned, assume the pattern is **not** found. Do **not** produce the item unless `:invert-match` was specified.

### Empty

Always produce the item. Even if `:invert-match` was specified. Shown in stats as a `passthru`.

### any other value

Produce that value.

PHASERS
-------

Any `FIRST`, `NEXT` and `LAST` phaser that are specified in the pattern `Callable`, will be executed at the correct time.

MATCHED ITEMS vs CONTEXT ITEMS
------------------------------

The `Pair`s that contain the search result within an object, have an additional method mixed in: `matched`. This returns `True` for items that matched, and `False` for items that have been added because of a context specification (`:context`, `:before-context`, `:after-context` or `paragraph-context`).

These `Pair`s can also be recognized by their class: `PairMatched` versus `PairContext`, which are also exported.

AUTHOR
======

Elizabeth Mattijsen <liz@raku.rocks>

Source can be located at: https://github.com/lizmat/rak . Comments and Pull Requests are welcome.

If you like this module, or what I’m doing more generally, committing to a [small sponsorship](https://github.com/sponsors/lizmat/) would mean a great deal to me!

COPYRIGHT AND LICENSE
=====================

Copyright 2022 Elizabeth Mattijsen

This library is free software; you can redistribute it and/or modify it under the Artistic License 2.0.

