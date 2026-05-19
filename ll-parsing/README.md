# Resilient LL Parsing, In Practice

This is a follow up of sorts to the [Resilient LL Parsing Tutorial](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html). It's a good article, but parts of it really didn't quite make sense at first until I actually tried to write something myself. The *why* of things wasn't always clear. So I'm writing down how I made my actual, fully functional, parser for a Rust-like language called Yagbas, and hopefully you'll understand why I did everything that I did.

No AI was used in the writing or coding of this article.

# Tokenizing The Source (aka "Lexing")

## Tokenizer Outline

Parsing is the process of taking disorganized data and making it progressively more organized so that we can work with it more easily. The first sub-step in that process is having a way to look in the source we're parsing for the little source code building blocks that make up the bigger program elements when combined properly. So we're looking for things like "the `fn` keyword" and "an identifier" and "an opening parentheses", which make up bigger things like "an entire function declaration" if they're all in the right order.

These smallest elements of the input are called a `Token` by convention. There are many kinds of token that exist, we've already thought about 3 of them, and when we find tokens they will *all* have a "span", which is the location in the source code file they came from. The span isn't important to the token itself, but it's critical when we're reporting errors. The user needs to know where the parsing went wrong.

We're going to have many kinds of token, each with a span. You might be tempted to write this:

```rust
#[derive(Debug, Clone, Copy)]
pub enum TokenKind {
  KwFn(Span),
  Ident(Span),
  OpParen(Span),
  // ... And so on
}
```

But when there's a common field to *all* variants of the enum, usually we want to lift that data out of the enum so that we don't have to match on the enum just to access it.

Instead we write it like this:

```rust
#[derive(Debug, Clone, Copy)]
pub struct Token {
  pub kind: TokenKind,
  pub span: Span,
}

#[derive(Debug, Clone, Copy)]
pub enum TokenKind {
  KwFn,
  Ident,
  OpParen,
  // ... And so on
}
```

Exactly what `TokenKind` variants exist depend on what you're parsing for, but we can have some guidelines that work for almost all situations:

* Every language keyword is a kind of token.
* Every element that varies in the source, like different identifiers and literal values, is a kind of token.
* Every punctuation is a kind of token.
* When in doubt, make *smaller* tokens and then handle any ambiguity later on in the parsing process.

What do I mean about ambiguity? Imagine there's `0.0` in the source, that's *obviously* a float literal... except when it's part of `x.0.0`. *Now* we'd like it to parse as accessing element 0 of element 0 of the variable `x`. This means we don't want `x.0.0` to be a 3 token "Ident Dot FloatLit", we want a 5 token "Ident Dot NumLit Dot NumLit" in our token sequence. And there's other examples, like `&&` being "boolean and" when it's an operator on its own, but in the code `&&x` we would generally want to instead read it as a reference to a reference to the variable `x`. The tokenizer should always tokenize the *least* bit of meaning it can, and then later steps of parsing can figure out more when the wider context is clearer.

We will have "a whole bunch of" `Token` values per source file, so we'd like each `Token` to be as small as possible. This means that we want `TokenKind` to be as small as possible, so we don't put any data within the variants. If it's a fixed token like a particular keyword or punctuation, then there's no useful data to add anyway. If it's a varying kind like an Ident, then the `Token` that wraps the `TokenKind::Ident` value will have a span for where to look in the source to find what the actual identifier was. Either way, the `TokenKind` value doesn't need any extra data.

Similarly, we want the `Span` type to be as small as possible. This part depends on if your language can mix tokens from different files inside a single syntax item. If that's possible, probably because some sort of macro system, then each span needs to have a way to mark what file, as well as where in that file.

```rust
pub type Span = (FileID, usize, usize);
```

Otherwise, you don't need the file identifier in each span, because you can store it as part of the overall syntax construction or error that gets parsed. You just need the start and end indexes:

```rust
pub type Span = (usize, usize);
```

The exact details of the Span type, using a type alias of a tuple, or your own structure or whatever, aren't really a big deal. As long as the span can be used to identify the original position in a source file, it can be whatever you want.

## Tokenizer Implementation

Now that we know what the output data of the tokenizer should look like, we need to "do that somehow". That *somehow* is commonly with the [logos](https://docs.rs/logos) crate, which we'll be using too.


