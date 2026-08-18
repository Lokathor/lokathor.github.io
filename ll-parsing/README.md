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
before doing the "actual" parser, and you'd basically be right. It isn't *necessary* to do lexing as a separate step, but there are enough advantages that it's a safe default.

So what is a token anyway? A token is the biggest thing that can be absolutely identified without any context at all. In other words, given a string slice of complete tokens, we can determine what tokens it has without looking at what tokens came before or after that slice. Since the Yagbas language is intended to "look like Rust" as much as possible, we can use some examples from Rust.

* If there is some source `x` that's an Identifier. There's lots of possible Identifier values like `foo` or `main` or `String`. It's how we name, or identify, things like functions and types and modules.
* If there is a `&` that's an Ampersand. If there are two in a row like `&&` that usually means the "Conditional And" operator, but it might also just mean a reference to a reference. Since we can't say for sure then we don't want the lexer to try thinking about it at all. During lexing, we just have the lexer emit two Ampersand tokens in a row, and the parser will think about it later.
* Similarly, if we have `0.0` it is tempting to just say that it's a floating point literal value. However, what if it was `x.0.0`? Oh, well in that case we probably want to have it be a field access of a field access of the `x` value. There's two ways to solve this:
    * A lexer that turns `0.0` into a LiteralFloat, and then a parser that allows float literals to be used as field accesses and then converts them into two separate int access field steps.
    * A lexer that turns `0.0` into a "LitNumber, Dot, LitNumber" sequence, and then a parser that can combine such a sequence into a float literal when looking for expressions. I prefer this route, so that the parser stage is just "combining" as often as possible, and not sometimes "separating".

So what does our `Token` type specifically look like in our code? We need many kinds of token, and they all have a span. We might at first think of writing this:

```rust
pub enum Token {
  LitNumber(Span),
  Ident(Span),
  Dot(Span),
  // etc
}
```

This isn't actually ideal for our needs, because to get the span we have to match on the variant, even though all variants have a span. Let's try something else:

```rust
pub struct Token {
  pub kind: TokenKind,
  pub span: Span,
}

pub enum TokenKind {
  LitNumber,
  Ident,
  Dot,
  // etc
}
```

Here we've simply "lifted" the common data into a struct that holds a `kind` field as well as other fields for the common data. This is great, we want to do this as much as we can.

The span info is not relevent to the tokens themselves, it's just important for error messages.
* If your tokens **can't** get mixed from different sources then the span can just be a `core::range::Range<usize>` or similar. Since there's only one source being processed at a time you naturally know what source the range is referring to.
* If your tokens **can** mix between sources (eg: because of a macro system like Rust has), then your span would have to be a `Range<usize>` and also a `FileId` or similar value that tracks which specific source the range refers to.

Now that we know how we want the output to look, we need to write a function something like this:

```rust
pub fn tokenize(src: &str) -> impl Iterator<Item=Token> + '_ {
  todo!()
}
```

We could probably accept something more generic as our input, like an iterator of bytes, but it would be less clear how the user should call the function. Generally the source data will be read from disk as a `String` value, so just accept `&str` for simplicity.

We could also return a `Vec<Token>` instead of an iterator, but it "feels right" to me to keep it as an iterator and allow the caller to actually allocate a Vec if that's what they want.

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
