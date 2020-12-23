# @safe by default

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Atila Neves (atila.neves@gmail.com)                             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Currently, `@system` is the default for unannotated functions. This DIP proposes
changing the default to `@safe`.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

Memory safety has been the origin of some of the most costly and
pernicious software defects ever known. D is a statically typed
language that aims to prevent bugs at compile-time, so it stands to
reason it should attempt to minimize memory safety defects.  It is
currently possible to do this with `@safe` but it not being
the default means it is not used as often as it could.

Most D code is technically `@safe` even if not annotated. This means that it is
currently hard or impossible to get the benefits of compiler-guaranteed
memory safety when using 3rd party libraries due to it being more effort to
add an annotation than to not do it.

Language defaults are important, and `@system` code should be relatively rare.


## Prior Work
* Memory-safe languages that offer a way to author system code such as C# and Rust.
* [DIP1028](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1028.md).
* A previous draft proposal: [@safe-by-default First Draft](https://github.com/dlang/DIPs/pull/153).

## Description

Functions with explicit `@safe`, `@trusted`, or `@system` annotations will not be
affected, nor will functions that have their attributes inferred
such as template functions, lambdas, and nested functions. Functions for which
attributes are inferred without an explicit annotation will behave as they do now.

Functions with a body visible to the compiler lacking a safety annotation will
now be `@safe`. Currently they are `@system`.

It will be a compiler error to declare a function with no body without
one of the three safety annotations. That is, such functions would no
longer have a safety default and must be explicitly annotated. While
mangling would cause linker errors when there are mismatches between a
declaration and a definition for `extern(D)` functions, memory safety
violations could occur in `extern(C)` or `extern(C++)` code.

Since it is likely that previously compiling code will fail to compile
with this DIP, the functionality will be opt-in and enabled with the
`-preview=safe` compiler switch.

No grammar changes are required.

The difference between this DIP and
[DIP1028](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1028.md)
is that function declarations (as opposed to definitions) have no
default safety level and must have an explicit annotation.


## Breaking Changes and Deprecations

It is likely that code will break, since unannotated `@system` functions will fail
to compile. This will require the maintainers to add the annotation to these
functions or change their implementation so that they are no longer `@system`.

### Virtual functions and covariance

`@safe` functions can override `@system` functions but not the other way around.
With `@safe` being the default, child classes that override that function might
no longer be able to depending on the implementation.

## Reference

* A [recent study](https://langui.sh/2019/07/23/apple-memory-safety/) showed that 60-70% of
vulnerabilities in iOS and macOS are due to memory safety.

* According to Microsoft, [70%](https://msrc-blog.microsoft.com/2019/07/18/we-need-a-safer-systems-programming-language/) of all vulnerabilities in their software have been caused by memory safety issues.

* More information about memory safety and why it's important can be found in
[this article](https://alexgaynor.net/2019/aug/12/introduction-to-memory-unsafety-for-vps-of-engineering/).


## Copyright & License
Copyright (c) 2021 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
The DIP Manager will supplement this section with a summary of each review stage
of the DIP process beyond the Draft Review.
