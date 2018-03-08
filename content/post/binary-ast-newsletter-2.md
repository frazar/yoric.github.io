+++
title = "The Binary AST Wishlist"
description = "Towards a JavaScript Binary AST, part 2"
tags = ["JavaScript", "Mozilla", "parsing", "ast", "compiler", "performance", "SpiderMonkey", "Babel", "newsletter"]
categories = ["JavaScript", "Mozilla", "performance", "binjs", "newsletter"]
draft = true
date = 2017-09-26T13:39:14+02:00
+++

In a previous post, we introduced the ongoing work on the [JavaScript Binary AST](https://yoric.github.io/post/binary-ast-newsletter-1/).
To recapitulate, the JavaScript Binary AST is a very special compression format designed:

- for JavaScript;
- to optimize end-to-end performance, in particular the time it takes for code to start executing.

This post attempts to explain some of the process behind the design of the Binary AST,
some of the choices we have already made and some of the choices that remain
to be made.

# Usability

Before entering in details, let's not forget the most important part: any piece
of technology, no matter how cleverly designed, is useless if nobody uses it.

So let's start our wish list with these points:

- **TODO** the result of this project must be useful;
- **TODO** the results of our work must be usable by everyone, without
    having to refactor or learn a new programming language or entire
    new toolchains;

# A short introduction to ASTs

AST stands for Abstract Syntax Tree. No matter which is your preferred
programming language, this data structure lurks inside your
interpreter/compiler/virtual machine. Even HTML has an AST,
called the DOM tree.

So, let us consider the following snippet of JavaScript:

```js
// `item` is declared implicitly
for (item of document.getElementsByClassName("sun")) {
    if (!item.classList.contains("clouds")) {
        item.classList.add("sundown");
    }
}
```

Between the time the browser receives the code and the instant it can start
executing it, this code needs to go through many (costly) steps.

After a few of these steps, the above source code is converted into something
like:

```js
ForOf {
    vars: ["item"],
    left: Name "item",
    right: Apply {
        function: Dot {
            object: Name "document",
            property: Name "getElementsByClassName"
        },
        args: [String "sun"]
    }
    body: If {
        condition: Unary {
            operation: "!",
            operand: Apply {
                function: Dot {
                    object: Dot {
                        object: Name: "item",
                        property: Name "classList"
                    },
                    property: Name "contains"
                },
                args: [String "clouds"]
            }
        }
        then: Apply {
            function: Dot {
                object: Dot {
                    object: Name: "item",
                    property: Name "classList"
                },
                property: Name: "add"
            },
            args: [String "sundown"]
        }
        else: null
    }
}
```

As you can see, this data structure contains the source code, just in a
different form. Once the compiler/interpreter/VM has created the AST,
it will perform a number of transformations on the AST, typically to either
detect errors or perform early optimizations.

After this, it will proceed to the next step of execution.

As you may guess from the name, the core stuff of the JS Binary AST project
is to ship a version of the JavaScript AST directly to the browser, instead
of the usual source code. While the above snippet looks large, we'll discuss
below and in future posts how we can actually make the AST smaller – indeed,
in most cases, much smaller than the (gzipped) source code.

# Coming up with an AST for JavaScript

The first problem is to define an AST for JavaScript. Indeed, while each
Virtual Machine has *an* AST, to this date, no two VM vendors have attempted
to work out a common JavaScript AST. This makes sense, because each VM does
different things with its AST and therefore stores different information on
it.

So, our first TODOs are:

- **TODO** we need a vendor-neutral AST;
- **TODO** we need to convince VM vendors that it can easily and efficiently be translated to their VM's internal AST;
- **TODO** this AST must be adopted as a JavaScript standard
- **TODO** whenever we specify a new JavaScript feature, its AST
    representation must also be specified.

As it turns out, there are already a few JavaScript ASTs in the wild:

- [ESTree](https://github.com/estree/estree)
- [Shift](http://shift-ast.org/)
- [Babylon](https://github.com/babel/babylon/blob/master/ast/spec.md)

We have decided to start our work based on Babylon, as it seems to be the most
battle-tested of the lot. However, our AST will have some deviations from
Babylon, where it makes sense for performance or file size.

## Yesterday's AST, today's AST and tomorrow's AST

If you have been using JavaScript recently, you have probably realized that the
language keeps changing. This means that yesterday's AST might not be today's
AST or tomorrow's AST.

For instance, consider a function. In ES5, a function was characterized by:

- optionally, its name;
- its list of arguments;
- its body.

So, a simple function

```js
function empty() {
    // Empty body.
}
```

was described as

```js
FunctionDeclaration {
    name: "empty"
    arguments: [], // Empty arguments.
    body: [], // Empty body.
}
```

In 2015, generators appeared, so that same function was described as

```js
FunctionDeclaration {
    name: "empty"
    arguments: [],
    body: [],
    generator: false,
}
```

In 2017, async functions appeared so that same function is now described as

```js
FunctionDeclaration {
    name: "empty"
    arguments: [],
    body: [],
    generator: false,
    async: false,
}
```

Of course, to make things more complicated, there is no guarantee that all VM
vendors implemented generators or async functions in the same order.

Similarly, new operators such as `**` or `**=` have been added to JavaScript,
as well as entirely new constructions such as `await`. In other words,
JavaScript is very much a moving target.

This translates to the following TODOs:

- **TODO** the AST format must be able to represent AST nodes
    that didn't exist when the AST format was defined;
- **TODO** the AST format must be able to represent AST nodes with
    properties that didn't exist when the AST format was defined;
- **TODO** the AST format must be able to represent AST nodes with
    values that didn't exist when the AST format was defined;
- **TODO** a VM implementing a more recent version of JavaScript must
    be able to read and process a BinAST that was emitted for an older
    version of JavaScript;
- **TODO** the AST format must not rely upon the order in which features
    have been proposed/adopted/implemented.

We fulfill these TODOs by adopting a self-describing format, which describes
inside the file which variant of a node is used. For instance, we may specify
something along the lines of:

```js
/*40*/ ...
/*41*/ ...
/*42*/ "FunctionDeclaration" ["name", "arguments", "body"]
/*43*/ ...
/*44*/ ...
```

This description lets the parser immediately determine which version(s) of
`FunctionDeclaration` may appear in the file and how they will be represented.
Here, the file contains `FunctionDeclaration` nodes with three
properties: `name`, `arguments` and `body`.

# Making it fast to send

If we want to make the Binary AST useful, we need to make sure that files do not
become too large. Not only that, but we would like Binary AST compression
to beat minification + gzip compression in terms of disk usage.

- **TODO** adopt a size-efficient representation of the AST;
- **TODO** during development, benchmark size against minified + gzipped code;

Of course, to do this, we need to adopt an efficient representation of the AST.

In the examples above, we have put the definition of `FunctionDeclaration` in an
index, with number 42. Which means that istead of

```js
FunctionDeclaration {
    name: "empty"
    arguments: [],
    body: [],
}
```

we can write in our file the following sequence of 9 bytes:

```js
[42, 5 /*length of string*/, 'e', 'm', 'p', 't', 'y', 0 /* length of list*/, 0 /* length of list */]
```

This is already much shorter.

Similarly, let us revisit

```js
// `item` is declared implicitly
for (item of document.getElementsByClassName("sun")) {
    if (!item.classList.contains("clouds")) {
        item.classList.add("sundown");
    }
}
```

We describe it as


```js
Constructions [
    /*0*/ "ForOf" ["vars", "left", "right"],
    /*1*/ "Name" ["value"],
    /*2*/ "Apply" ["function", "args"],
    /*3*/ "Dot" ["object", "property"],
    /*4*/ "If" ["condition", "then", "else"],
    /*5*/ "Unary" ["operation", "operand"],
    /*6*/ "String" ["value"],
    /*7*/ null,
]
Strings [
    /*0*/"item",
    /*1*/"document",
    /*2*/"getElementsByClassName",
    /*3*/"sun",
    /*4*/"classList",
    /*5*/"contains",
    /*6*/"clouds",
    /*7*/"add",
    /*8*/"sundown",
    /*9*/"!",
]

ForOf /*0*/{
    vars: ["item"], /*1, 0*/
    left: Name "item"  /*1, 0*/,
    right: Apply /*2*/{
        function: Dot /*3*/{
            object: Name "document" /*1, 1*/,
            property: Name "getElementsByClassName" /*1, 2*/
        },
        args: [String "sun"] /*1, 6, 3*/
    }
    body: If /*4*/{
        condition: Unary /*5*/{
            operation: "!", /*9*/
            operand: Apply /*2*/{
                function: Dot /*3*/{
                    object: Dot /*3*/{
                        object: Name: "item", /*1, 9*/
                        property: Name "classList" /*1, 4*/
                    },
                    property: Name "contains" /*1, 5*/
                },
                args: [String "clouds"] /*1, 6, 6*/
            }
        }
        then: Apply /*2*/{
            function: Dot /*3*/{
                object: Dot /*3*/{
                    object: Name: "item", /*1, 9*/
                    property: Name "classList" /*1, 4*/
                },
                property: Name: "add" /*1, 7*/
            },
            args: [String "sundown"] /*1, 6, 8*/
        }
        else: null /*7*/
    }
}
```

Or, in binary...

```js
/* Index*/ ...
/* Strings */ [/*Number of strings*/10, /*Length*/4, 'i', 't', 'e', 'm', /*Length*/8, 'd', 'o', 'c', 'u', 'm', 'e', 'n', 't', /*Length*/22, 'g', 'e', 't', 'E', 'l', 'e', 'm', 'e', 'n', 't', 's', 'B', 'y', 'C', 'l', 'a', 's', 's', 'N', 'a', 'm', 'e', /*Length*/3, 's', 'u', 'n', /*Length*/ 9, 'c', 'l', 'a', 's', 's', 'L', 'i', 's', 't', /*Length*/8, 'c', 'o', 'n', 't', 'a', 'i', 'n', 's', /*Length*/8, 'c', 'l', 'o', 'u', 'd', 's', /*Length*/ 3, 'a', 'd', 'd', /*Length*/ 7, 's', 'u', 'n', 'd', 'o', 'w', 'n'
/* AST */ [0, 1, 0, 1, 0, 2, 3, 1, 1, 1, 2, 1, 6, 3, 4, 5, 9, 2, 3, 3, 1, 9, 1, 4, 1, 5, 1, 6, 6, 2, 3, 3, 1, 9, 1, 4, 1, 7, 1, 6, 8, 7]
```

- **TODO** use well-known compression techniques to further reduce the size of
    the binary.

# Making it fast to parse

If you recall, the main problem we are trying to solve is performance. More
precisely, we want JavaScript code to *start* faster. Smaller files will
definitely help (more on this below), but we can go much further.

Before we proceed, let me offer you a game: how quickly can you read and understand
the following snippet?

```txt
function /*function() {}*/ replace() { // }
   var string /* my string */ = /.* my string */;
   var re = /* out of the re */ } " /* in the re */ ";
   (...x) => ({string});
`}`
```

(yeah, I have deactivated syntax coloring on purpose, I'm that kind of guy)

I hope that, after playing this game, you agree with me that the syntax of
JavaScript is complicated and understanding it needs way too much brainpower.
For fun, you can try and find how to reactivate syntax coloring – and realize
that the syntax coloring tool actually misinterprets this source code, or
perhaps I do.

So, here's something we can do:

- **TODO** the syntax of the AST must be extremely simple for the computer to parse;

And in particular

- **TODO** don't parse things more than once;

But that's not all. Parsing JavaScript wouldn't be so much of a problem if we
didn't actually need to parse *the entire file* before we could start executing
it.

- **TODO** the syntax of the AST and semantics of parser must allow us to *skip*
    over stuff that may not be useful;

So far, we have attempted to decrease the total amount of work before we can
start, which is generally the best way to do things. The alternative is to
delegate it to other cores:

- **TODO** the syntax of the AST and semantics of parser must allow us to
    delegate some of the work to other nodes.
