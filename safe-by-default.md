# @safe by default

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Átila Neves (atila dot neves at gmail)                          |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

Make `@safe` the default instead of `@system`.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

While `@safe` makes it possible to write memory-safe code in D and
have the compiler verify that this is indeed the case, having it be
opt-in makes it hard to use in practice. Third-party libraries in
particular, Phobos included, will limit the usage of `@safe` unless an
effort is made to make their code callable from `@safe` functions.

Most D functions are `@safe` in practice, however, and defaults
matter.  Making `@safe` be the default will enable more D code to be
verifiably memory-safe. Given that most functions are effectively
`@safe`, implementing this DIP will also reduce the number of
necessary annotations in D code.

## Prior Work

[DIP1028](https://github.com/atilaneves/DIPs/blob/master/DIPs/rejected/DIP1028.md).

[A previous draft proposal](https://github.com/dlang/DIPs/pull/153).

## Description

Currently functions whose attributes are not inferred and that do not
have an explicit `@safe`/`@trusted`/`@system` attribute are assumed to
be `@system`. This DIP proposes that the default become `@safe`. Since
this is a breaking change it will be gated by the
`-preview=safedefault` compiler switch.

This DIP makes no distinction between `extern(D)`, `extern(C)`, and
`extern(C++)` functions. Although the latter two *usually* apply to
functions written in C or C++ respectively, that is not necessarily
the case. Callback functions can be implemented in D, and thereby
correctly proven to be `@safe` by the compiler. The `extern`
declarations only affect mangling and calling convention, and not what
programming language they were written in.

This DIP does however make a distinction between declarations and
definitions; the latter refers to the case when the function's body is
visible to the compiler. In this case, `@safe` will be the default.
If there is no body, the compiler cannot verify the `@safe`ty of the
function and in those cases this DIP proposes that there will be no
default. All declarations (i.e. functions with no body) must have
exactly one of the `@safe`/`@trusted`/`@system` annotations or the
module that contains them will fail to compile. This will handle the
common case of declaring functions that are implemented in another
language.


## Breaking Changes and Deprecations

This will likely break a lot of code that was not previously annotated
since `@safe` functions have restrictions that do not apply to
`@system` functions. Functions that use features not allowed in
`@safe` functions such as taking the address of a local will no longer
compile. Virtual functions that override a function that is now
`@safe` will not be able to be `@system`.

## Reference

* The DIP1028 [formal assessment discussion](https://forum.dlang.org/post/rwjxbgsauknjjrvousti@forum.dlang.org).
* The DIP1028 [final review discussion](https://forum.dlang.org/post/jelbtgegkwcjhzwzesig@forum.dlang.org).
* The DIP1028 [final review feedback](https://forum.dlang.org/post/jelbtgegkwcjhzwzesig@forum.dlang.org).

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
