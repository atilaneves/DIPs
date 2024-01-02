# String Interpolation

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Review Count:   | 0 (edited by DIP Manager)                                       |
| Author:         | Átila Neves atila dot neves at gmail                            |
| Implementation: | https://github.com/dlang/dmd/pull/15715                         |
| Status:         | Draft                                                           |

## Abstract

Allow embedding expressions within strings for purposes such as string
formatting, easier code generation, logging, and much more by taking
advantage of D's superior compile-time metaprogramming features.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale
Required.

A short motivation about the importance and benefits of the proposed change.  An existing,
well-known issue or a use case for an existing projects can greatly increase the
chances of the DIP being understood and carefully evaluated.

## Prior Work
Required.

If the proposed feature exists, or has been proposed, in other languages, this is the place
to provide the details of those implementations and proposals. Ditto for prior DIPs.

If there is no prior work to be found, it must be explicitly noted here.

## Description

An _interpolated string_ is any D string literal prefixed with the
letter `i`, as in `i"Hello"`, and will be referred to as _i-strings_
in the remainder of the document. D has several kinds of string
literals such as `r"WYSIWYG strings"` and `q{token strings}` but for
the sake of simplicity this document will only concern itself with
_double quoted i-strings_, i.e. `i"Here go characters"`.

### Example

The i-string `i"foo $bar $(baz + 4) ok"` is lowered into following
[compile-time sequence](https://dlang.org/articles/ctarguments.html):

```d
(
    InterpolationHeader(),
    InterpolatedLiteral!"foo "(),
    InterpolatedExpression!"bar"(),
    bar,
    InterpolatedLiteral!" "(),
    InterpolatedExpression!"baz + 4"(),
    baz + 4,
    InterpolatedLiteral!" ok"()
    InterpolationFooter()
)
```

The types used above would all be defined in `core.interpolation`.
Under the present proposal, the assertion in the following code would
hold:

```d
import std.conv: text;
auto foo = "a string";
struct MyStruct { int member; }
auto bar = Struct(42);
auto baz = 2;
assert(text(i"foo $bar $(baz + 4) ok") == "foo MyStruct(42) 6 ok");
```

A literal `$` can be obtained by escaping it with an additional `$`:

```d
assert(text(i"I have $$5.00 in my account") == "I have $5.00 in my account");
```

### Syntax and gramar

This document proposes changing the [existing D grammar] with the following
diff:


```diff
PrimaryExpression:
    Identifier
    . Identifier
    TemplateInstance
    . TemplateInstance
    this
    super
    null
    true
    false
    $
    IntegerLiteral
    FloatLiteral
    CharacterLiteral
    StringLiteral
+   InterpolatedString
    ArrayLiteral
    AssocArrayLiteral
    FunctionLiteral
    AssertExpression
    MixinExpression
    ImportExpression
    NewExpression
    FundamentalType . Identifier
    ( Type ) . Identifier
    ( Type ) . TemplateInstance
    FundamentalType ( NamedArgumentListopt )
    TypeCtor ( Type ) . Identifier
    TypeCtor ( Type ) ( NamedArgumentListopt )
    Typeof
    TypeidExpression
    IsExpression
    ( Expression )
    SpecialKeyword
    TraitsExpression

+InterpolatedString:
+    InterpolatedDoubleQuotedString
+
+InterpolatedDoubleQuotedString:
+    i" InterpolatedDoubleQuotedCharacters "
+
+InterpolatedDoubleQuotedCharacters:
+    InterpolatedDoubleQuotedCharacter InterpolatedDoubleQuotedCharacters
+
+InterpolatedDoubleQuotedCharacter:
+    Character
+    InterpolationSequence
+    InterpolationEscapeSequence
+    EndOfLine
+
+InterpolationEscapeSequence:
+    $$
+
+InterpolationSequence:
+    $( AssignExpression )
```


### Detailed Explanation

An "unescaped" or "non-escaped" `$` character is `InterpolationSequence`
as defined above in the grammar changes. Escaped `$` characters, i.e.
multiple occurences in a row, will be considered part of the string with
one fewer `$`: `$$` becomes `$`, `$$$` becomes `$$`, and so forth.

i-strings are lowered to compile-time sequences by following these rules:

The first element is always `InterpolationHeader()`.

The last element is always `InterpolationFooter()`.

For the following elements, and starting at index 0, apply the
following algorithm:

If the current index is a non-escaped `$`, "emit" an
`InterpolationExpression` with the string template argument set to the
contents of the balanced parentheses following it, followed by a
variadic list of values that the expression evaluates to. This will
usually be one value but could be more in the case of
tuples/`AliasSeq`/nested i-strings. The current index will then be
advanced to the one past the `InterpolationExpression`, i.e. one past
the closing parenthesis.

Otherwise "emit" an `InterpolatedLiteral` with its string template
argument set to
`i-string[currentIndex .. indexOfNextInterpolatedExpression]`.
If there are no further `InterpolatedExpression`s, the end index of the
slice is `$`, i.e. the remainder.

If the first character is an unescaped `$`, the first non-header
element (i.e. the 2nd or index 1) is `InterpolationExpression!""`
where the string template argument is the contents of the
`InterpolatedExpression`. Otherwise, the first non-header element is
i-string[0 .. x] where `x` is the index of the nearest
`InterpolatedExpression`, i.e. the index of the corresponding `$`
character.

Any nested i-strings are recursively expanded.


### Runtime support

`core.interpolation` will contain the definition for
`InterpolationHeader`, `InterpolationFooter`, `InterpolatedLiteral`,
and `InterpolatedExpression`. All of these are guaranteed to have a
member function with the following declaration:


```d
string toString() @safe @nogc pure nothrow const
{
    return null;
}
```

This ensures that they transparently work with standard library
functions such as `writeln` or `text`.

`InterpolatedLiteral` and `InterpolatedExpression` will both have exactly
one string template argument.


## Breaking Changes and Deprecations

Since there are no current interpolated strings, no existing code will
break or need to be deprecatd.


## Reference

Optional links to reference material such as existing discussions,
research papers or any other supplementary materials.


## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)


## Reviews
