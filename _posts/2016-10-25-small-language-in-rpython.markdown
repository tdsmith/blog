---
title: "Implementing a small language in RPython"
layout: post
---

Inspired by Alex Gaynor's ["So You Want To Write An Interpreter?"](https://www.youtube.com/watch?v=LCslqgM48D4) PyCon talk, I decided I wanted to try to write an interpreter! In this post I'll talk about writing Baby's First Interpreter, porting my interpreter to RPython, and performing the translation.

My final code lives [in a Github repository](https://github.com/tdsmith/dummylang) if you want to follow along.

Like Alex in his talk, I'll use [rply](https://github.com/alex/rply) to generate the lexer and parser for my interpreter. Alex's [documentation for rply](https://rply.readthedocs.io/en/latest/) gives a walkthrough for building a simple calculator parser that generates an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) which can be evaluated by the interpreter.

Before continuing with this blog post, I recommend implementing and playing with the calculator demo to get a sense of how the lexer and parser interact and to get a sense for the feel of an AST class heirarchy.

Looking for a slightly taller hill to climb, I found [an old paper](/img/eeniemeeny.pdf) [[source]](http://dl.acm.org/citation.cfm?id=234880) by Lawrence A. Coon with iteratively more complex sample languages designed for a compiler design lab class. This post will be about my experience with his `eenie` language.

## Lexing eenie

I started writing the front-end for eenie by building the lexer. The `eenie` lexer is morally similar to the calculator lexer with a couple of key differences:

* `eenie` has identifiers: we need the lexer to generate tokens for the names of variables
* `eenie` has reserved words we need to identify, like `program`

Handling reserved words is a little tricky with `rply`'s lexer generator because you need to be able to distinguish a reserved word like `has` from a legal identifier name like `hasty`. If you create a rule like `lg.add("HAS", "has")`, the regex `has` will match `hasty`, which is not what you want. The pattern `"has$"` doesn't ever match.

`rply` is based on [PLY](http://www.dabeaz.com/ply/), so I went looking at `PLY`'s documentation; `PLY` essentially lets you define a callback function that's invoked each time a certain type of token is identified that lets you modify the lexer output. You can solve this problem with `PLY` by creating an identifier rule and then, in the callback, identify the identifiers that are actually reserved words, and then emit a new token type for them.

`rply` doesn't let us define callbacks but we can do something morally similar by intercepting the lexer output after it's produced and operating on the output a token at a time before feeding it to the parser:

```python
reserved = ["program", "has", "decls", ...]
lg.add("ID", r"[a-zA-Z][a-zA-Z0-9]*")
lexer = lg.build()

...

def lex(corpus):
	for token in lexer.lex(corpus):
		if token.name == "ID" and token.value.lower() in reserved:
			yield Token(token.value.upper(), token.value)
		else:
			yield token
```

Note that it's possible to test the lexer without implementing a parser!

## Parsing eenie to an AST

The grammar in Appendix A of the `eenie` paper contains the production rules that we can feed the parser generator to produce our parser. The calculator parser is, again, a good example to begin with.

### BNF notation

One notation note is that the BNF for `eenie` contains rules like:

`a ::= a, b | b`

which produces an `a` from either a `b`, or an existing `a` plus a `b`. While translating to rply, in order to generate a product from more than one rule, define an additional generator rule instead of using an or-pipe, like:

```python
@pg.production("a : a COMMA b")
def a_from_ab(p):
	pass

@pg.production("a : b")
def a_from_b(p):
	pass
```

You can also apply more than one production rule to a single method, like in Alex's calculator example:

```python
@pg.production('expression : expression PLUS expression')
@pg.production('expression : expression MINUS expression')
@pg.production('expression : expression MUL expression')
@pg.production('expression : expression DIV expression')
def expression_binop(p):
	pass
```

Note that the canonical `eenie` grammar specifies separate `term`, `factor`, and `exp` products in order to handle operator precedence. You can use `exp` for all of these if you add [predence rules](https://rply.readthedocs.io/en/latest/api/rply.html#rply.ParserGenerator) for the arithmetic operators to the ParserGenerator constructor.

### Building up a parser

I found it helpful to begin by implementing the parser in stages.

* Declare the production rules and methods but just return the input `p` from all of them. Now, you can make sure that a) parser generation (`parser = pg.build()`) works, and b) the parser runs on your example code.
* Implement the AST classes and modify the production methods to return instances of those classes.
* Implement .eval() on the AST classes.

### Evaluating the AST

One feature `eenie` has, which the calculator language is missing, is variable assignment and reference. A simple way to keep track of these variables is to pass a Python dictionary around as you evaluate the AST which contains the current values of all defined variables.

Therefore, the signature of the eval function is `ASTNode.eval(self, context)` where `context` is a `dict` mapping strings (identifier names) to `Number`s (variable values). Variable assignment stores a value in `context` and identifier references evaluate to the matching value in `context`.

### High five!

My "finished" Python interpreter is here: [https://github.com/tdsmith/dummylang/tree/master/eenie](https://github.com/tdsmith/dummylang/tree/master/eenie).

## Porting eenie to RPython

I quickly discovered a number of incompatibilities between RPython and my Python coding style. :) RPython is a [subset of Python](http://rpython.readthedocs.io/en/latest/rpython.html) upon which type inference is possible. The advantage of writing an interpreter in RPython is that the RPython translation toolchain can compile your interpreter and make it Very Fast. The disadvantage is that RPython is much stricter than standard Python. Some of the restrictions are:

### Type stability

* Functions should always be called with arguments of the same type (except that functions that accept complex types may also accept None)

  This means, for example, that you can't lazily cast something to an `int` after it's been passed into a constructor. You always need to give the constructor an `int`!
  
* Functions should always return a value of the same type, except that a) functions that return complex types may also return None, and b) instances of different classes that inherit from the same superclass are allowed.

* Names must hold variables of a single type in any given scope; i.e. `a = "foo"; a = Bar()` is not allowed.

For the purpose of this paragraph, a "complex type" is any type that is not an integer, float, or boolean.

The things I needed to change about my naïve Python implementation were:

* Stop using [`attrs`](https://attrs.readthedocs.io/en/stable/) for my AST classes. :(
* Make sure that all production functions returned a descendent of the same superclass; mine was called `ASTNode`. In particular, they cannot return `list`s; I had to jury-rig a `ASTList` class. (While you implement this, note that RPython doesn't support the `__add__` dunder on user-defined classes.)
* Make sure that `ASTNode.eval()` always returns an `ASTNode` or `None`, and never a primitive.

Adding and checking static type annotations with [mypy](http://mypy-lang.org/) is a great intermediate step before trying to translate your interpreter with RPython; the mypy error messages are considerably more cromulent.

### Standard library support

The only stdlib modules that have RPython support are `math`, `os`, and `time`. This means that operations on `sys.stdin` are right out; `raw_input` is also unimplemented. Instead, use `os.read(0, ...)` and `os.write(1, ...)` to interact with stdin and stdout.

### Defining a translation target

The RPython translation framework needs some hints about how to invoke your interpreter. [This blog post](https://morepypy.blogspot.com/2011/04/tutorial-writing-interpreter-with-pypy.html#pypy-translation) has a useful sample that I stole wholesale.

### Perform the translation

Download a pypy source tarball or a hg checkout somewhere and run:

`python ~/pypy/rpython/bin/rpython target.py`

If all goes well, you will have a fancy compiled interpreter in `./target-c` in about 30 seconds!

All did not go well for me. I began getting `UnionError`s which reflected that I had called methods using different type signatures. Tracking all of these down took longer than I want to admit in such a small codebase. I think earlier use of `mypy` would have saved me a lot of angst. You can also just run `python target.py prog1.eenie` to make sure you should be expecting sane behavior.

### Profit???

My RPython eenie interpreter [lives here](https://github.com/tdsmith/dummylang/tree/master/eenie_rpython).

I hope this story about my experience was helpful! The RPython toolchain is a powerful tool for language implementors and I look forward to playing with it in greater detail.