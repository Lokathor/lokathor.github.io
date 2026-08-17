# Resilient LL Parsing, In Practice

This is a follow up of sorts to the [Resilient LL Parsing
Tutorial](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
Thats's a good article, but parts of it really did not make sense to me until I
read it several times and then implemented it myself, then it finally started to
turn the gears in my brain.

So what I'm going to do to try and improve the situation is describe the "why"s
and the "how" of the Resilient LL parser that I wrote for the
[Yagbas](https://github.com/lokathor/yagbas) language, which is a language I've
been tinkering with for a few years. I will be describing the full, working
version of the language's parsing system, without leaving anything out as an
exercise to the reader.

You should not need to have read the previous article to understand this one. I
may refer to it in a few places as a point of comparison, but everything here is
intended to make sense on its own.

No AI was used in the writing or coding of this article.

# Tokenizing The Source (aka "Lexing")

## Tokenizer Goals

The parser has to turn totally unstructured text into as well structured a
format as it can. A common design is to have an initial phase that just breaks
the source code string into "tokens", then the main parser just works on the
token values. You might think that this sounds like doing a separate parser
before doing the "actual" parser, and you'd basically be right.

For a lot of langauges, the main advantage you get is that once you've turned
the source string into source tokens then the rest of the parser doesn't have to
think about whitespace at all. That's certainly enough for me.

## Using `logos` to Tokenize

TODO

## Writing a Tokenizer By Hand

TODO

# Resilient Parsing

TODO

## Parser Concepts

TODO

## Parser Building Blocks

TODO

## Parser Implementation

TODO
