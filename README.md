# String Interpolation

| Field           | Value                                                             |
|-----------------|-------------------------------------------------------------------|
| DIP:            | xxxx                                                              |
| Review Count:   | 0                                                                 |
| Author:         | Andrei Alexandrescu<br>John Colvin jcolvin@symmetryinvestments.com|
| Implementation: |                                                                   |
| Status:         |                                                                   |

## Abstract

Instead of requiring a format string followed by an argument list or interspersing of format fragments with other arguments, string interpolation enables embedding the arguments in the string itself. We propose an extremely simple yet powerful approach of lowering interpolated strings into compile-time tuples (akin to `AliasSeq` in the standard library). We demonstrate how this approach achieves all major objectives of an interpolated strings feature with a minimal footprint on the language definition and support library.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Copyright & License](#copyright--license)

## Rationale

A frequent pattern in programming is to create strings (for the purpose of printing to the console, writing to files, etc) by mixing predefined string fragments with data contained in variables. The most straightforward approach to implement that pattern is to intersperse expressions with string fragments in calls to variadic functions:

```d
void f1(string name) {
    writeln("Hello, ", name, "!");          // formats and prints in one go
    string s = text("Hello, ", name, "!");  // formats and returns formatted string
    ...
}
```

A more flexible approach, embodied by the classic `printf` family of C functions, is to use *format specifiers* that provide the string fragments intermixed with formatting directives. Specialized functions take such format specifiers alongside data components, and replace each formatting directive with suitably formatted data:

```d
void f2(string name) {
    writefln("Hello, %40s!", name);           // formats and prints in one go
    string s = format("Hello, %40s!", name);  // formats and returns formatted string
    ...
}
```

Such an approach observes the important principle of [separating logic from display](https://www.cs.usfca.edu/~parrt/papers/mvc.templates.pdf). This principle is well respected in a variety of programming paradigms, such as Model-View-Controller, web development, and UX design. Localization and internationalization applications and libraries can store all display artifacts in complete separation from program logic and swap them as needed. In a perfect separation model, there is no access to computation in the formatting strings at all, even as much as a simple addition or (in the case of `printf` format strings) even the names of the variables being printed.

However, there are important use cases where separation of data from format is not only unneeded, but becomes a hindrance:

- *code generation:* when generating code, the tight integration between the string fragments and the interspersed computation is essential to the process;
- *scripting:* shell scripts use string interpolation very frequently, to the extent that the mechanics of quoting and interpolation is an essential focus of all shell scripting languages;
- *casual printing, tracing, logging, and debugging:* sometimes, such tasks have more focus on the expressions to be printed, than on formatting paraphernalia.

Such needs are served poorly by either the interspersion approach and the format specifier approach. Consider this example adapted from the implementation of `std.bitmanip.bitfields`:

```d
enum result = text(
    "@property bool ", name, "() @safe pure nothrow @nogc const { return ",
    "(", store, " & ", maskAllElse, ") != 0;}\n",
    "@property void ", name, "(bool v) @safe pure nothrow @nogc { ",
    "if (v) ", store, " |= ", maskAllElse, ";",
    "else ", store, " &= cast(typeof(", store, "))(-1-cast(typeof(", store, "))", maskAllElse, ");}\n"
);
```

(The [original code](https://github.com/dlang/phobos/blob/v2.095.1/std/bitmanip.d#L115) uses string concatenation instead of a call to `text`. We use the latter to simplify the example.) Here, the interspersion mechanics distract from following the correctness of the generated code. An approach based on format specifiers would look as follows:

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

This form has less syntactic noise and appears as a format string separated from the expressions involved, for which reason we afforded to reformat it in a shape consistent with the generated code. Correctness of the generated code is still difficult to assess on the generated code, for example the reader must mentally map tedious sequences such as `%3$s` to meaningful names such as `maskAllElse` throughout the code snippet.

By comparison, using the interpolation syntax proposed in this DIP would make the code much easier to follow:

```d
enum result = text(
    i"@property bool {name}() @safe pure nothrow @nogc const {
        return ({store} & {maskAllElse}) != 0;
    }
    @property void {name}(bool v) @safe pure nothrow @nogc {
        if (v) {store} |= {maskAllElse};
        else {store} &= cast(typeof({store}))(-1-cast(typeof({store})){maskAllElse});
    }\n"
);
```

The latter form has dramatically less syntactic noise and appears as a single string with expressions inside escaped by `{` and `}`. Correctness of the generated code is much easier to assess in the second form as well.

Let us also look at a shell command example. Assume `url` is an URL and `file` is an filename, both preprocessed as escaped shell strings. To download `url` into `file` without risking corrupt files in case of incomplete downloads, the code below first downloads into a temporary file with the extension `.frag` and then atomically renames it to the correct name:

```d
executeShell("wget " ~ url ~ " -O" ~ file ~ ".frag && mv " ~ file ~ ".frag " ~ file);
```

The version using format specifiers is marginally more readable:

```d
executeShell("wget %1$s -O%2$s.frag && mv %2$s.frag %2$s", url, file);
```

The interpolated form is, again, by far the easiest to follow:

```d
executeShell(i"wget {url} -O{file}.frag && mv {file}.frag {file}");
```

Last but not least, there are numerous cases in which casual console output can use interpolated strings to reduce on boilerplate and improve clarity:

```d
writeln("Hello, ", name, ". You are ", age, " years old.");  // interspersion
writefln("Hello, %s. You are %s years old.", name, age);     // format string
writeln(i"Hello, {name}. You are {age} years old.");         // interpolation
```

### Why Yet Another String Interpolation Proposal?

This DIP derives from, and owes much to, the previous work on string interpolation in the D community. The abundance of such work raises the question of why a new proposal is needed.

This DIP is close to the prior work yet different in key aspects as follows:

- Like [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988), this DIP expands the interpolated string into an argument list. However, unlike that proposal that automatically passes the expansion as an argument list for `std.typecons.tuple`, we place the expansion in a built-in tuple. We will show how using expansion to built-in tuple has significant flexibility and efficiency advantages.
- Like [DIP 1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), this DIP expands the interpolated string into a built-in tuple. Unlike DIP 1027, our proposal does not assemble the string fragments together, has a simpler syntax, and does not foster a format-string-style approach. We will argue that unifying format strings with string interpolation is not a goal worth pursuing. We will also show how custom formatting can be elegantly implemented on the library side with no additional language support, no complication of the syntax, and no loss of efficiency.
- We heed important lessons from [DIP 1036](https://github.com/dlang/DIPs/blob/master/DIPs/DIP1036.md), mainly that while pursuing generality, complexity must be kept under control. In wake of it, we argue that it is not only appropriate, but in fact recommendable, to abandon certain directions of generalization in favor of a drastic reduction in complexity. In particular, we make integration with format-string style approaches a non-goal.

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

An interpolated string is a regular D string prefixed with the letter `i`, as in `i"Hello"`. No whitespace is allowed between `i` and the opening quote.

An interpolated string may occur only in one of the following contexts:

- in the argument list of a function call;
- in the argument list of a constructor call;
- in the argument list of a `mixin`;
- in the argument list of a template instantiation.

In any other context, interpolated strings are ill-formed.

For example, the function call expression:

```d
writeln(i"I ate {apples} and {bananas} totalling {apples + bananas} fruit.")
```

is lowered into:

```d
writeln("I ate ", apples, " and ", bananas, " totalling ", apples + bananas, " fruit.");
```

The resulting lowered code is subjected to the usual typechecking and has the same semantics as if the lowered code were present in the source.

After introducing an intuition of how interpolated string work, let us formalize the syntax and semantics. Lexically:

```
InterpolatedString:
   i" DoubleQuotedCharacters "
```

The `InterpolatedString` appears in the parser grammar as an `InterpolatedExpression`, which is under `PrimaryExpression`.

```
InterpolatedExpression:
   InterpolatedString
   InterpolatedString StringLiterals

StringLiterals:
   StringLiteral
   StringLiteral StringLiterals
```

Inside an interpolated string, the characters `{` and `}` are of particular interest because the interpolated string will use them as escapes. To make `{` and `}` printable inside an interpolated string, the sequences `{{` and `}}`, respectively, shall be used. The contents of the `InterpolatedExpression` must conform to the following grammar:

```
Elements:
    Element
    Element Elements

Element:
    Character excluding '{' and '}'
    '{{'
    '}}'
    '{' Identifier '}'
    '{' Type '}'
    '{' Expression '}'
```

In the grammar above `Type` is the nonterminal for types, and `Expression` is the nonterminal for general D expressions.

The `InterpolatedExpression` is converted to a comma-separated list that consists of the string fragments interspersed with the expressions escaped by the characters `{` and `}`.

Any lexical errors (such as unbalanced `{` and `}`) will be reported during parsing. Semantic checking will ensure that interpolated strings occur only in the contexts specified above. Then, other semantic errors will be reported during the typechecking of the lowered code.

This concludes the syntax and semantics of the proposed feature.

### Use Cases

Although the proposed feature is deceptively simple, its flexibility affords a multitude of use cases. They can be easily assessed with a current D compiler by typing in the lowered code.

#### Passing arguments to functions

The simplest use case of interpolated strings, and the most likely to be encountered in practice, is as argument to functions such as `writeln`:

```d
void main(int argc, string[] argv) {
    import std.stdio;
    writeln(i"The program {argv[0]} received {argc - 1} arguments.");
    // Lowering: --->
    // writeln("The program ", argv[0], " received ", argv.length - 1, " arguments.");
}
```

A function such as `writeln` above has no way to know whether it was called via an interpolated string or with interspersed arguments; interpolated strings are purely a call-side device. We consider this a key characteristic of the feature.

#### Saving the result of interpolation as a tuple

Although it may seem limiting to impose that interpolated strings expand in a function call, `tuple` offers an immediate and efficient mechanism for storing the result of interpolation. The following program produces the same output as the previous one:

```d
void main(int argc, string[] argv) {
    import std.stdio;
    auto t = tuple(i"The program {argv[0]} received {argc - 1} arguments.");
    // Lowering: --->
    // auto t = tuple("The program ", argv[0], " received ", argv.length - 1, " arguments.");
    writeln(t.expand);
}
```

#### Custom formatting

It may seem that this proposal has a fundamental limitation: what if we want to do some custom formatting on the arguments, such as displaying an integral number in hexadecimal or a floating-point number in scientific format?

Fortunately, an elegant solution can be implemented on the library side with minimal effort. Consider for example defining a `print` function that supports a *formatting directive* called `hex` that instructs the function to print the integral that follows in hexadecimal. First, the library defines a `hex` constant with a unique type:

```d
struct Hex {}
immutable Hex hex;
```

The library recognizes arguments of type `Hex` as directives to print the next integral in hexadecimal format. On the call side, the user simply inserts `hex` in the interpolated strings appropriately when calling `print`:

```d
void fun(int x) {
    print(i"{x} in hexadecimal is 0x{hex}{x}.");
    // Lowering: --->
    // print(x, " in hexadecimal is 0x", hex, x, ".");
}
```

There is no need for defining, implementing, and memorizing a mini-language of encoded format specifiers --- everything can be done with plain D values. The approach is reminiscent of [C++ stream manipulators](http://www.cplusplus.com/reference/library/manipulators/), thankfully with a much lower syntactical load.

For another example, suppose the library sets out to support custom formatting for floating-point numbers, such as width, precision, and scientific notation. It would first define a small data structure that contains the appropriate state:

```d
struct Fixed { uint width, precision; }
Fixed fixed(uint width = 10, uint precision = 6) {
    return Fixed(width, precision);
}

struct Scientific { dchar sep; }
Scientific scientific(dchar sep = 'E') {
    return Scientific(sep);
}
```

The formatting library recognizes `Fixed` and `Scientific` as formatting directives controlling formatting, allowing the client to pass them just like any arguments:

```
double x = 0.1 + 0.2;
print(i"{x} can be written as {scientific}{x} and its exact value is {fixed(20, 10)}{x}");
// Lowering: --->
// print(x, " can be written as ", scientific, x, " and its exact value is ", fixed(20, 10), x);
```

#### Use in `mixin` declarations and expressions

Interpolated strings are allowed in `mixin` declarations and expressions:

```d
immutable x = "asd", y = 42;
mixin(i"int {x} = {y};");
// Lowering --->
// mixin("int ", x, " = ", y, ";");
auto z = mixin(i"{x} + 5");
// Lowering --->
// auto z = mixin(x, " + 5");
```

#### Use in the argument list of template instantiations

An interpolated string may be present in a template instantiation's argument list. This allows, for example, a parser generator to compose with strings in the grammar definition:

```d
struct Grammar(string... spec) { ... }

immutable ident = "( [ char ]+ )"

alias Calculator = Grammar!(
    i"Expression := Term + Term
    Term := Factor * Factor
    Factor := {ident} | '(' Expression ')' "
);
// Lowering: --->
// alias Calculator = Grammar!(
//     "Expression := Term + Term
//     Term := Factor * Factor
//     Factor := ", ident, " | '(' Expression ')' "
// );
```

In certain cases, the interpolated string can be passed to `AliasSeq` as well resulting in a sequence of strings interspersed with identifiers:

```d
int x = 42;
alias Q = AliasSeq!(i"I'm interpolating {x} here.");  // OK, use Q as a type
auto q = AliasSeq!(i"I'm interpolating {x} here.");   // OK, store as AliasSeq object
```

#### Conversion to format-string-style arguments

TODO

### Limitations and tradeoffs

Users may be confused that they cannot define variables that are interpolated strings:

```d
int x = 42;
auto s = i"Let's interpolate {x}!"  // Error, interpolated string not allowed here
```

The remedy is simple and may be suggested by the text of the error message: use a tuple to store the interpolation, or call a function such as `text` to convert everything to a string:

```d
int x = 42;
auto s = text(i"Let's interpolate {x}!");        // OK, store as string
auto t = tuple(i"Let's interpolate {x}!");       // OK, store as tuple
alias Q = AliasSeq!(i"Let's interpolate {x}!");  // OK, use as type
auto q = AliasSeq!(i"Let's interpolate {x}!");   // OK, store as AliasSeq
```

Functions are not aware whether they got called with an interpolated string or a manually written list of arguments. This confers consistency, simplicity, and uniformity to the approach.

## Breaking Changes and Deprecations

Because `InterpolatedString` is a new token, no existing code is broken.

## Copyright & License

Copyright (c) 2021 by the D Language Foundation

Licensed under Creative Commons Zero 1.0

