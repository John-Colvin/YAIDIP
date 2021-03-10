# String Interpolation

| Field           | Value                                                             |
|-----------------|-------------------------------------------------------------------|
| DIP:            | xxxx                                                              |
| Review Count:   | 0                                                                 |
| Author:         | Andrei Alexandrescu<br>John Colvin jcolvin@symmetryinvestments.com|
| Implementation: |                                                                   |
| Status:         |                                                                   |

## Abstract

Instead of requiring a format string followed by an argument list, string interpolation enables
embedding the arguments in the string itself.


## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

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

Such an approach observes the important principle of [separating logic from display](https://www.cs.usfca.edu/~parrt/papers/mvc.templates.pdf) . This principle is well respected in a variety of programming paradigms, such as Model-View-Controller, web development, and UX design. Localization and internationalization applications and libraries can store all display artifacts in separation from program logic and swap them as needed.

However, there are important use cases where separation of logic from display is not only unneeded, but becomes a hindrance:

- *code generation:* when generating code, the tight integration between the string fragments and the interspersed computation is essential to the process;
- *scripting:* shell scripts use string interpolation very frequently, to the extent that the mechanics of quoting and interpolation is an essential focus of all shell scripting languages;
- *casual printing, tracing, logging, and debugging:* sometimes, such tasks have more focus on the expressions to be printed, than on formatting paraphernalia.

Such needs are served poorly by either the interspersion approach and the format specifier approach Consider this example adapted from the implementation of `std.bitmanip.bitfields`:

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

By comparison, using a hypothetical interpolation syntax would make the code much easier to follow:

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

## Prior Work

* Interpolated strings have been implemented and well-received in many languages.
For many such examples, see [String Interpolation](https://en.wikipedia.org/wiki/String_interpolation).
* [DIP1027](https://github.com/dlang/DIPs/blob/master/DIPs/rejected/DIP1027.md), from which this DIP was derived.
* [C#'s implementation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated#compilation-of-interpolated-strings) which returns a formattable object that user functions can use
* [Javascript's implementation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals) which passes `string[], args...` to a builder function very similarly to this proposal
* Jason Helson submitted a DIP [String Syntax for Compile-Time Sequences](https://github.com/dlang/DIPs/pull/140).
* [Jonathan Marler's Interpolated Strings](http://github.com/dlang/dmd/pull/7988)

## Description

TODO
