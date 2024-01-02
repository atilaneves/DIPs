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

[String
interpolation](https://en.wikipedia.org/wiki/String_interpolation) has
been implemented in multiple languages, examples being:
* [Javascript's template
literals](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)
, where user-defined functions can be passed the string fragments and
values similarly to this proposal.
* [Python's
  f-strings](https://docs.python.org/3/tutorial/inputoutput.html#tut-f-strings).
* [C#](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated),
  where users can access a formattable object.

There has also been previous work on string interpolation for D:
* [Jonathan Marler's interpolated strings](http://github.com/dlang/dmd/pull/7988).
* [Jason Helson's pull request](https://github.com/dlang/DIPs/pull/140).
* [DIP1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md).
* [DIP1036](https://github.com/dlang/DIPs/blob/master/DIPs/other/DIP1036.md).
* [YAIDIP](https://github.com/John-Colvin/YAIDIP).

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

### Syntax and grammar

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

## Use Cases

This section discusses potential ways to use the language feature
described in this document.  It does *not* mandate any library
implementations nor suggests adding the examples presented to Phobos.

### What already works

By construction, string interpolation as presented in this document
would work
["out-of-the-box"](https://github.com/adamdruppe/interpolation-examples/blob/master/01-basics.d)
with existing Phobos functions such as `std.stdio.writeln` and
`std.conv.text`:

```d
import std;
string[string] AA = ["hello":"world"];
writeln(i"The AA has \"$(AA["hello"])\" inside it."); // The AA has "world" inside it.
int i;
string s = i"i: $(i)".text;
```

### Custom formatting

This proposal does not directly address custom formatting in the style
of C's `printf` or D's `std.format.format`, but it does make it
possible for a
[library](https://github.com/adamdruppe/interpolation-examples/blob/master/lib/format.d)
to be written (or code to be added to Phobos) that [makes it
possible](https://github.com/adamdruppe/interpolation-examples/blob/master/02-formatting.d):

```d
string name = "D user";
float wealth = 55.70;

// only $ followed by a ( is special.
// so the double $ here is a basic one followed by an
// interpolated var. Can also use \$ in i"strings"
// then :% is interpreted *by this function* to mean
// "use this format string for the preceding argument"
writefln(i"$(name) has $$(wealth):%0.2f");
```

The exact syntax of the mini-language used for formatting would be
decided by the library authors.

### `printf` compatibility

Although D has alternative and superior ways of formatting strings, it
may be that some users would like to pass interpolated strings into
`printf`. As in the previous case of custom formatting, this is not
directly addressed by the language feature being proposed, but a
[support
library](https://github.com/adamdruppe/interpolation-examples/blob/master/lib/printf.d)
can be written that [enables
compatibility](https://github.com/adamdruppe/interpolation-examples/blob/master/03-printf.d)
with `printf`, `sprintf`, etc.:

```d
int age = 3;
string name = "The child";
printf(makePrintfArgs(i"$(name) is $(age) years old.\n").tupleof);
```

### Internationalization

The flexibility of this string interpolation proposal makes it
possible to use [gettext](https://en.wikipedia.org/wiki/Gettext#),
along with some [support
code](https://github.com/adamdruppe/interpolation-examples/blob/master/04-internationalization.d)
to translate interpolated strings into different languages:

```d
int coffees = 5;
int iq = -30;

writeln(tr(i"You drink $(coffees) cups a day and it gives you $(coffees + iq) IQ"));
// You drink 5 cups a day and it gives you -25 IQ
gettext.selectLanguage("mo/german.mo"); // change the language to German
writeln(tr(i"You drink $(coffees) cups a day and it gives you $(coffees + iq) IQ"));
// Sie trinken 5 Tassen Kaffee am Tag. Dadurch erhöht sich Ihr IQ auf -25
```

### SQL prepared statements

With an i-string aware
[library](https://github.com/adamdruppe/interpolation-examples/blob/master/lib/sql.d)
one can create prepared SQL statements with values embedded
[directly](https://github.com/adamdruppe/interpolation-examples/blob/master/06-sql.d):

```d
int id = 1;
string name = "' DROP TABLE', '";
db.execi(i"INSERT INTO sample VALUES ($(id), $(name))");

foreach(row; db.query("SELECT * from sample"))
writeln(row[0], ": ", row[1]);
// Prints (no SQL injection):
// 1: ' DROP TABLE', '
```

### HTML generation

Again, with [supporting
code](https://github.com/adamdruppe/interpolation-examples/blob/master/lib/html.d),
one can write D i-strings with values embedded within them that get
[correctly
escaped](https://github.com/adamdruppe/interpolation-examples/blob/master/07-html.d)
for consumption by a browser:

```d
string name = "<bar>"; // this will be properly encoded despite the angle brackets
auto element = i"<foo>$(name)</foo>".html; // a dom.d Element
assert(element.tagName == "foo"); // a structured object, not a string

writeln(element.toString());
// <foo>&lt;bar&gt;</foo>
```

## Breaking Changes and Deprecations

Since there are no current interpolated strings, no existing code will
break or need to be deprecatd.


## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)


## Reviews
