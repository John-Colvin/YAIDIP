# String Interpolation

| Field           | Value                                                             |
|-----------------|-------------------------------------------------------------------|
| DIP:            | xxxx                                                              |
| Review Count:   | 0                                                                 |
| Author:         | Andrei Alexandrescu<br>John Colvin john.loughran.colvin@gmail.com |
| Implementation: |                                                                   |
| Status:         |                                                                   |

## Abstract

Textual formatting is often achieved by APIs relying either on format specification strings followed by arguments to be formatted (in the style of `printf`, `std.format.format`, and `std.stdio.writefln`), or on interspersing arguments of string and non-string types (in the style of `std.conv.text` and `std.stdio.writeln`). String interpolation enables a style of formatting that embeds arguments within a literal string. We propose an extremely simple yet powerful approach of lowering interpolated strings into comma-separated lists that works with both *format-string* style and *interspersion* style with no or minimal changes to functions to take advantage of such interpolated strings. We demonstrate how this approach achieves all major objectives of an interpolated strings feature with a minimal footprint on the language definition and support library.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Copyright & License](#copyright--license)

## Rationale

A frequent pattern in programming is to create strings (for the purpose of printing to the console, writing to files, etc) by mixing predefined string fragments (literal strings) with data contained in variables. We identify two distinct approaches to formatting data: *interspersion* style and *format-string* style.

The most straightforward approach is to intersperse expressions with string literals in calls to variadic functions:

```d
void f1(string name) {
    writeln("Hello, ", name, "!");          // formats and prints in one go
    string s = text("Hello, ", name, "!");  // formats and returns formatted string
    ...
}
```

A more flexible approach, embodied by the classic `printf` family of C functions and carried over to D standard library functions such as `std.format.format` and `std.stdio.writefln`, is to use *format strings* that contain conventionally defined *format specifiers*. Specialized functions take such format strings followed by the arguments to be formatted and replace each format specifier with suitably formatted data:

```d
void f2(string name) {
    writefln("Hello, %40s!", name);           // formats and prints in one go
    string s = format("Hello, %40s!", name);  // formats and returns formatted string
    ...
}
```

Other examples of the format-string style are string templates for formatting HTML documents and SQL prepared statements. The convention used for format specifiers is defined by the respective APIs:

```d
void f2(string name) {
    htmlOutput("Looking for #{}...", name);                  // specifier is #{}
    auto rows = sql("SELECT * FROM t WHERE name = ?", name); // specifier is a question mark
    ...
}
```

Each approach --- interspersion style and format-string style --- has its pros and cons. The format-string style observes the important principle of [separating logic from display](https://www.cs.usfca.edu/~parrt/papers/mvc.templates.pdf). This principle is well respected in a variety of programming paradigms and domains, such as Model-View-Controller, UX design, and web development. Localization and internationalization applications and libraries can store all display artifacts in complete separation from program logic and swap them as needed. In a perfect separation model, there is no access to computation in the formatting strings at all, even as much as a simple addition or (in the case of `printf` format strings) even the names of the variables being printed. The disadvantage of the format-string style is that the expressions to be formatted appear lexically *separate from* the (possibly long) format string, which makes it difficult to follow which format specifiers correspond to their respective arguments.

The interspersion style is simple, intuitive, and requires learning no convention. However, creating complex outputs becomes cumbersome due to the syntactic heaviness of alternating string literals and other arguments in comma-separated lists. Also, customized formatting (such as rendering an integral in hexadecimal instead of decimal) is not supported.

Below we provide evidence to the difficulties of both the format-string style and the interspersion style for three categories of typical tasks:

- *code generation:* when generating code, the tight integration between the string literal fragments and the expressions to be inserted is essential to the process;
- *scripting:* shell scripts use string interpolation very frequently, to the extent that the mechanics of quoting and interpolation is an essential focus of all shell scripting languages;
- *casual printing, tracing, logging, and debugging:* often, such tasks have more focus on the expressions to be printed, than on separating formatting paraphernalia from the data to be formatted.

The following examples show that such needs are served poorly by either the interspersion approach and the format-string approach. Consider an example of code generation adapted from the implementation of `std.bitmanip.bitfields`:

```d
enum result = text(
    "@property bool ", name, "() @safe pure nothrow @nogc const { return ",
    "(", store, " & ", maskAllElse, ") != 0;}\n",
    "@property void ", name, "(bool v) @safe pure nothrow @nogc { ",
    "if (v) ", store, " |= ", maskAllElse, ";",
    "else ", store, " &= cast(typeof(", store, "))(-1-cast(typeof(", store, "))", maskAllElse, ");}\n"
);
```

(The [original code](https://github.com/dlang/phobos/blob/v2.095.1/std/bitmanip.d#L115) uses string concatenation instead of a call to `text`. We use the latter to simplify the example.) Here, the interspersion mechanics (closing quote, comma, expression, comma, opening quote) distract the reader from following the correctness of the generated code. An approach based on format specifiers would look as follows:

```d
enum result = format(
    "@property bool %s() @safe pure nothrow @nogc const {
        return (%s & %s) != 0;
    }
    @property void %s(bool v) @safe pure nothrow @nogc {
        if (v) %s |= %s;
        else %s &= cast(typeof(%s))(-1-cast(typeof(%s))%s);
    }\n",
    name, store, maskAllElse,
    name, store, maskAllElse, store, store, store, maskAllElse
);
```

This form has less syntactic noise and appears as a format string separated from the expressions involved, for which reason we afforded to reformat it in a shape consistent with the generated code. However, the separation of format from data is an impediment here requiring the reader to mentally track and pair the format specifiers `%s` with the arguments trailing the formatting string. The repetition of arguments after the formatting string is also problematic. Using positional arguments brings a marginal improvement to the code because the arguments must be passed only once and referred by their position:

```d
enum result = format(
    "@property bool %1$s() @safe pure nothrow @nogc const {
        return (%2$s & %3$s) != 0;
    }
    @property void %1$s(bool v) @safe pure nothrow @nogc {
        if (v) %2$s |= %3$s;
        else %2$s &= cast(typeof(%2$s))(-1-cast(typeof(%2$s))%3$s);
    }\n",
    name, store, maskAllElse
);
```

Here, the reader only needs to track the correct use of numbers in the format specifiers and match it with the order in the trailing arguments. Correctness of the generated code is still difficult to assess, for example the reader must mentally map tedious sequences such as `%3$s` to meaningful names such as `maskAllElse` throughout the code snippet.

By comparison, using the interpolation syntax proposed in this DIP would make the code much easier to follow:

```d
enum result = text(
    i"@property bool $name() @safe pure nothrow @nogc const {
        return ($store & $maskAllElse) != 0;
    }
    @property void $name(bool v) @safe pure nothrow @nogc {
        if (v) $store |= $maskAllElse;
        else $store &= cast(typeof($store))(-1-cast(typeof($store))$maskAllElse);
    }\n"
);
```

The latter form has dramatically less syntactic noise and appears as a single string with expressions inside escaped by `$`. Correctness of the generated code is much easier to assess as well.

Let us also look at a shell command example. Assume `url` is an URL and `file` is an filename, both preprocessed as escaped shell strings. To download `url` into `file` without risking corrupt files in case of incomplete downloads, the code below first downloads into a temporary file with the extension `.frag` and then atomically renames it to the correct name:

```d
executeShell("wget " ~ url ~ " -O" ~ file ~ ".frag && mv " ~ file ~ ".frag " ~ file);
```

The version using format specifiers is marginally more readable:

```d
executeShell("wget %s -O%s.frag && mv %s.frag %s", url, file, file, file);  // classic
executeShell("wget %1$s -O%2$s.frag && mv %2$s.frag %2$s", url, file);      // positional
```

The interpolated form is, again, the easiest to follow:

```d
executeShell(i"wget $url -O$file.frag && mv $file.frag $file");
```

Last but not least, there are numerous cases in which casual console output can use interpolated strings to reduce on boilerplate and improve clarity:

```d
writeln("Hello, ", name, ". You are ", age, " years old.");  // interspersion
writefln("Hello, %s. You are %s years old.", name, age);     // format string
writeln(i"Hello, $name. You are $age years old.");           // interpolation
```

### Why Yet Another String Interpolation Proposal?

This DIP derives from, and owes much to, the previous work on string interpolation in the D community. The abundance of such work raises the question why a new proposal is needed.

This DIP is close to the prior work yet different in key aspects as follows:

- Like [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988), this DIP expands the interpolated string into an argument list. However, unlike that proposal that automatically passes the expansion as an argument list for `std.typecons.tuple`, we expand into an argument list and leave the rest to user code. We will show how doing so has significant flexibility and efficiency advantages.
- Like [DIP 1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), this DIP expands the interpolated string into an argument list. Unlike DIP 1027, which only supports the format-string style, this proposal supports both the interspersion and the format-string style. It also has a simpler syntax and semantics.
- We heed important lessons from [DIP 1036](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1036.md), mainly that while pursuing generality, complexity must be kept under control.

We will demonstrate how this DIP achieves all major goals of extant proposals with a radically simpler definition and implementation.

## Related Work

* Interpolated strings have been implemented and well-received in many languages.
For many such examples, see [String Interpolation](https://en.wikipedia.org/wiki/String_interpolation).
* [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988) and [DIP1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), from which this DIP was derived.
* [DIP1036](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1036.md), which is as of the time of this writing in review.
* [C#'s implementation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated#compilation-of-interpolated-strings) which returns a formattable object that user functions can use
* [Javascript's implementation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) which passes `string[], args...` to a builder function very similarly to this proposal
* Jason Helson submitted a DIP [String Syntax for Compile-Time Sequences](https://github.com/dlang/DIPs/pull/140).

## Description

D strings have several syntactical forms, among which are those that follow the pattern of a letter followed by the string proper. Such include `r"WYSIWYG strings"`, `q"[delimited strings]"`, and `q{token strings}`. Our proposal follows the same pattern by introducing `i"interpolated strings"`.

An *interpolated string* is a regular D string prefixed with the letter `i`, as in `i"Hello"`. No whitespace is allowed between `i` and the opening quote. We refer to these constructs as `i`-strings.

An `i`-string is allowed in source code only in one of the following contexts:

- in the argument list of a function or constructor call;
- in the argument list of a string `mixin`;
- in the argument list of a `pragma(msg)` directive;
- in the argument list (starting with the second argument) of an `assert` or `static assert` invocation, contingent to fixing [Issue 17378](https://issues.dlang.org/show_bug.cgi?id=17378); and
- in the argument list of a template instantiation.

In any other context, `i`-strings are illegal.

For an example of `i`-strings usage, the function call expression:

```d
writeln(i"I ate $apples apples and $bananas bananas totalling $(apples + bananas) fruit.")
```

is lowered into:

```d
writeln(__header, "I ate ", apples, " apples and ", bananas, " bananas totalling ", apples + bananas, " fruit.")
```

The `__header` value, to be discussed later, is generated by the compiler and contains compile-time information about the interpolated string.

The resulting lowered code is subjected to the usual typechecking and has the same semantics as if the lowered code were present in the source.

After introducing an intuition of how interpolated string work, let us formalize the syntax and semantics. Lexically:

```
InterpolatedString:
   i" DoubleQuotedCharacters "
```

The `InterpolatedString` appears in the parser grammar as an `InterpolatedList`, which is under `ArgumentList`.

```
ArgumentList:
    AssignExpression
    AssignExpression ,
    AssignExpression , ArgumentList
    InterpolatedString
    InterpolatedString ,
    InterpolatedString , ArgumentList
```

Inside an interpolated string, the character `$` is of particular interest because the interpolated string will use it as an escape. If `$` is not followed by an open paren or an identifier, its meaning is unchanged. If `$` is followed by an identifier (which starts with a `_` or alphabetic character) or open parenthesis, the `$` acts as an escape character introducing an interpolated identifier or expression. To render `$` verbatim when followed by an identifier or an open paranthesis, the sequence `$$` shall be used. The contents of the `InterpolatedExpression` must conform to the following grammar:

```
Elements:
    Element
    Element Elements

Element:
    Character excluding '$'
    '$$'
    '$' Character other than '$', '_', 'a'-'z', or 'A'-'Z'
    '$' Identifier
    '$(' Type ')'
    '$(' AssignExpression ')'
```

In the grammar above `Type` is the nonterminal for types, and `AssignExpression` is the nonterminal for general D assignment expressions. For details refer to the current grammar at https://dlang.org/spec/grammar.html.

The `InterpolatedString` is lowered to a comma-separated list that consists of a header followed by the string fragments interspersed with the expressions escaped by `$`.

Any lexical errors (such as unbalanced `(` and `)`) will be reported during parsing. Then, semantic errors will be reported during the typechecking of the lowered code.

### Normalization of interspersion

The lowering is normalized such that:

(a) A `__header` object is always the first element in the resulting argument list;
(b) The rest of the resulting argument list always starts with a string literal;
(c) The argument list always ends with a string literal; and
(d) The argument list always alternates between literal strings and expressions, i.e. there are never two consecutive literal strings or two consecutive expressions.

To effect these rules, the lowering may introduce empty strings as follows.

If an interpolated string starts with an escape sequence, an empty string is always introduced before it in the expansion. Example:

```D
writeln(i"$name, hi!");
```

is lowered to:

```D
writeln(__header, "", name, " hi!");
```

If an interpolated string ends with an escape sequence, an empty string is always introduced after it in the expansion. Example:

```D
writeln(i"Hello, world$exclamation");
```

is lowered to:

```D
writeln(__header, "Hello, world", exclamation, "");
```

Finally, if an `i`-string contains two consecutive expansions, the lowering will introduce an empty string literal in between. Example:

```D
writeln(i"Hello, $name$exclamation How are you?");
```

is lowered to:

```D
writeln(__header, "Hello", name, "", exclamation, " How are you?");
```

These rules can apply simultaneously on the same `i`-string. Example:

```D
writeln(i"$greeting, $name$exclamation");
```

is lowered to:

```D
writeln(__header, "", $greeting, ", ", name, "", exclamation, "");
```

The purpose of normalization, as can be seen in the given examples, is to always have the argument list in a fixed pattern: *header*, *string-literal*, *expression*, *string-literal*, *expression*, ...,  *string-literal*.

## The Interpolation Header

Every `i`-string lowering introduces a header rvalue object that so far we conventionally denoted as `__header` (the exact name is uniquely compiler-generated and hence inaccessible). The header is a stateless `struct` that contains exclusively compile-time information about the interpolation, generated from the following template:

```D
struct InterpolationHeader(_parts...) {
    alias parts = _parts;
    string toString() { return null; }
}
```

The argument to the instantiation of `Header` is the interpolation string deconstructed into parts and normalized. Example:

```D
writeln(i"$greeting, $name$exclamation");
```

is lowered to:

```D
writeln(InterpolationHeader!("", "greeting", ", ", "name", "", "exclamation", "")(),
    "", $greeting, ", ", name, "", exclamation, "");
```

Note how the header gives the callee complete access to the strings corresponding to the expressions passed in.

Due to normalization, `parts` always has an odd number of elements. Strings at even indices in `parts` always originate in the literal fragments of the interpolated string. Strings at odd indices are always string representations of D expressions.

The header object has a trivial `toString` method that expands to the `null` string. This method makes it possible to pass interpolated strings directly to functions such as `writeln` and `text` because these functions detect and use `toString` to convert unknown data types to strings. More sophisticated functions that do want to detect interpolated strings can detect the presence of `Header` objects by simple introspection.

This concludes the syntax and semantics of the proposed feature.

### Discussion: The Choice of Escape Grammar

Why choose the `$` when many popular languages and libraries (Python, C++20, C#) use `{` and `}` as escape characters? Also, why use `$(` and `)` as opposed to `${` and `}`, as perhaps a bash user may be more familiar with?

One essential use of interpolation, specific to D, is for code generation. In generated D code, curly braces `{` and `}` are abundant. Requiring `{{` and `}}` everywhere in the generated code would have been aggravating. Anecdotal evidence has been collected in the creation of this DIP, which initially attempted to use `{` and `}` for escaping: the examples extracted from `std.bitmanip.bitfields` turned out to have numerous bugs, and be difficult to read when corrected:

```d
// Using `{` and `}` for escaping, similar to Python's f-strings
enum result = text(
    i"@property bool {name}() @safe pure nothrow @nogc const {{
        return ({store} & {maskAllElse}) != 0;
    }}
    @property void {name}(bool v) @safe pure nothrow @nogc {{
        if (v) {store} |= {maskAllElse};
        else {store} &= cast(typeof({store}))(-1-cast(typeof({store})){maskAllElse});
    }}\n"
);
```

In contrast, occurrences of `$` followed by an open parenthesis or an identifier in D code are rare --- indeed most likely to be present in generated code that uses interpolation itself. This makes `$` a disproportionately strong candidate compared to other choices.

The second question --- why not use `${` and `}` instead of `$(` and `)` --- has a simple answer: the elements to group inside the escape sequences are expressions, not statements. There already exists a syntax for grouping expressions, and that's surrounding them with `(` and `)`, which closes the case by invoking the [principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).

## Use Cases

Although the proposed feature is deceptively simple, its flexibility affords a multitude of use cases. They can be easily assessed with a current D compiler by typing in the lowered code.

### Passing arguments to functions

The simplest use case of interpolated strings, and the most likely to be encountered in practice, is as argument to functions such as `writeln`, `text`, `writefln`, or `format`:

```d
void main(string[] args) {
    import std.stdio;
    writeln(i"The program $(args[0]) received $(args.length - 1) arguments.");
    // Lowering: --->
    // writeln(InterpolationHeader!("The program ", "args[0]", " received ", "args.length - 1", " arguments.")(),
    //     "The program ", args[0], " received ", args.length - 1, " arguments.");

    auto s = sqlExec(i"INSERT INTO runs VALUES ($(args[0]), $(args.length - 1))");
    // Lowering: --->
    // auto s = sqlExec(InterpolationHeader!("INSERT INTO runs VALUES(", "args[0]", ", ", "args.length - 1", ")")(),
    //     "INSERT INTO runs VALUES(", args[0], ", ", args.length - 1, ")");
}
```

A function such as `std.stdio.writeln` may choose to uniformly convert the header to string by means of calling its `toString` method, thus essentially working with interpolated strings without modification. The second possibility is that the function detects the header but "skips" it and ignores the information it provides, which is easy to accommodate on the function implementer's side. Finally, a function may choose to fully support `i`-strings with specialized semantics. We consider this flexibility a key characteristic of the proposed feature that drastically simplifies both its definition, understandability, and interoperation with new and existing code.

A format-string-style function such as `writefln` does not work unchanged with interpolated strings because the lowering is unhelpful:

```D
writefln(i"Hello, %s$name %s$surname!");
// Lowering: --->
// writefln(InterpolationHeader!("Hello, %s", "name", " %s", "surname", "!")(),
//     "Hello, %s", name, " %s", surname, "!");
```

However, there is enough information in the interpolated string to allow `writefln` to work transparently with interpolated strings by detecting and processing during compilation the header information:

```D
// Possible semantics: format spec precedes argument
writefln(i"Hello, %s$name %s$surname!");
// Result:
// "Hello, John Smith!"
```

#### Saving the result of interpolation as a tuple

Although it may seem limiting to impose that interpolated strings expand to an argument list (most often in a call to a user-provided function call), `tuple` offers an immediate and efficient mechanism for storing the result of interpolation. The following program produces the same output as the previous one:

```d
void main(string[] args) {
    import std.stdio;
    auto t1 = tuple(i"The program $(args[0]) received $(args.length - 1) arguments.");
    // Lowering: --->
    // auto t1 = tuple(InterpolationHeader!("The program ", "args[0]", " received ", "args.length - 1", " arguments.")(),
    //    "The program ", args[0], " received ", args.length - 1, " arguments.");
    writeln(t1.expand);
}
```

With the existing implementation, `tuple` will save the interpolation header as its first data member. Of course, it is easy to adjust its implementation to drop it if deemed necessary.

#### Manipulator-style formatting

[C++'s iostreams](http://www.cplusplus.com/reference/iolibrary/) introduced the notion of [*stream manipulators*](https://en.cppreference.com/w/cpp/io/manip) --- special functions interspersed with the data to print, which direct the way data is to be formatted. For example, the C++ `std::dec` and `std::hex` manipulators instruct the formatting engine to format the following integer in decimal and hexadecimal, respectively:

```C++
// C++ code
#include <iostream>
void fun(int x) {
    std::cout << std::dec << x << " in hexadecimal is 0x" << std::hex << x << ".\n";
}
```

The approach is obviously extensible by simply adding new manipulators. Unfortunately, C++ stream manipulators have developed a poor reputation because they are syntactically heavy --- interspersion is done with the visually prominent `<<` and the user must choose between polluting their namespace and using the `std::` prefix for scope resolution with each manipulator. (These issues and other unrelated ones motivated the introduction of the `<format>` facility in C++20.)

Using interpolation, a manipulators-based approach can be used elegantly and implemented with minimal effort. Consider for example using stream manipulators such as `dec` and `hex` for `writeln` by using an `i`-string:

```D
void fun(int x) {
    writeln(i"$dec$x in hexadecimal is 0x$hex$x.");
    // Lowering: --->
    // writeln(InterpolationHeader!("", "dec", "", "x", " in hexadecimal is 0x", "hex", "", "x", ".")(),
    //     "", dec, "", x, " in hexadecimal is 0x", hex, "", x, ".");
}
```

There is no need for defining, implementing, and memorizing a *sui generis* mini-language of encoded format specifiers --- all formatting can be done with D language expressions. Continuing the example, the library can just as easily define parameterized formatting for floating-point numbers, such as width, precision, and scientific notation:

```d
void fun(double x) {
    writeln(i"$x can be written as $scientific$x or $(fixed(20, 10))$x.");
    // Lowering: --->
    // writeln(InterpolationHeader!("", "x", " can be written as ", scientific, "", x, " or ", fixed(20, 10), "", x, ".")(),
    //     x, " can be written as ", scientific, "", x, " or ", fixed(20, 10), "", x, ".");
}
```

#### Use in `mixin` declarations and expressions

`i`-strings are allowed in `mixin` declarations and expressions.

```d
immutable x = "asd", y = 42;
mixin(i"int $x = $y;");
// Lowering --->
// mixin(InterpolationHeader!("int ", "x", " = ", "y", ";")(),
//     "int ", x, " = ", y, ";");
auto z = mixin(i"$x + 5");
// Lowering --->
// auto z = mixin(InterpolationHeader!("", "x", " + 5")(),
//     "", x, " + 5");
```

The interpolation header is ignored by `mixin`s.

#### Use in `pragma(msg)` directives

`i`-strings are allowed in `pragma(msg)` directives:

```d
enum x = 42;
pragma(msg, i"x = $x.");
// Lowering --->
// pragma(msg, InterpolationHeader!("x = ", "x", ".")(),
//     "x = ", x, ".");
```

Note that `pragma(msg)` is already variadic. Currently `assert` and `static assert` are not variadic, so they need to be helped with `text` or `format`:

```d
void fun(int x)(int y) {
    static assert(x < 42, text(i"x is $x, should be less than 42"));
    assert(y > 42, format(i"y is %s$y, should be greater than 42"));
}
```

#### Use in the argument list of template instantiations

An interpolated string may be present in the  argument list of a template instantiation. This allows, for example, a parser generator to compose with strings in the grammar definition:

```d
struct Grammar(spec...) { ... }

immutable identifier = "( [ character ]+ )"

alias Calculator = Grammar!(
    i"Expression := Term + Term
    Term := Factor * Factor
    Factor := $identifier | '(' Expression ')' "
);
// Lowering: --->
// alias Calculator = Grammar!(InterpolationHeader!(...)(),
//     "Expression := Term + Term
//     Term := Factor * Factor
//     Factor := ", identifier, " | '(' Expression ')' "
// );
```

In certain cases, the interpolated string can be passed to `AliasSeq` as well resulting in a sequence of strings interspersed with identifiers:

```d
int x = 42;
alias p = AliasSeq!(i"I'm interpolating $x here.");  // p is a value sequence
auto q = AliasSeq!(i"I'm interpolating $x here.");   // q is an implicit tuple
```

### Limitations and tradeoffs

Users may be confused that they cannot define variables that are interpolated strings:

```d
int x = 42;
auto s = i"Let's interpolate $x!"  // Error, interpolated string not allowed here
```

The remedy is simple and may be suggested by the text of the error message: use a tuple to store the interpolation, or call a function such as `text` or `format` to convert everything to a string:

```d
int x = 42;
auto s = text(i"Let's interpolate $x!");        // OK, store as string
auto t = tuple(i"Let's interpolate $x!");       // OK, store as tuple
alias p = AliasSeq!(i"Let's interpolate $x!");  // OK, value sequence
auto q = AliasSeq!(i"Let's interpolate $x!");   // OK, store as an implicit tuple
```

## Breaking Changes and Deprecations

Because `InterpolatedString` is a new token, no existing code is broken.

## Copyright & License

Copyright (c) 2021 by the D Language Foundation

Licensed under Creative Commons Zero 1.0
