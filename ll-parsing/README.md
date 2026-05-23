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

Let's imagine a new file called `tokenizer.rs`, it has our `Token` definition just like before.

```rust
// tokenizer.rs

use logos::Logos;

#[derive(Debug, Clone, Copy)]
pub struct Token {
  pub kind: TokenKind,
  pub span: Span,
}
```

We'll need the `TokenKind` and `Span` types too of course. In this case, we're making a `Span` type that can be `Copy`, but we'll also still use it to index into our source string frequently later on, so we'll make a little converter function for getting a `Range<usize>`.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Logos)]
pub enum TokenKind {
  // TODO
}

#[derive(Debug, Clone, Copy)]
pub struct Span {
  pub start: usize,
  pub end: usize,
}
impl Span {
  pub fn as_range(self) -> core::ops::Range<usize> {
    self.start..self.end
  }
}
```

We're going to derive the `Logos` trait on `TokenKind`. Setting this up correctly has many steps. Each step is relatively small and simple, but there's still a lot of them. 

First step is to add logos as a dependency:

```sh
lokathor@corvid:~/code/yagbas$ cargo add logos --features=forbid_unsafe
    Updating crates.io index
      Adding logos v0.16.1 to dependencies
             Features:
             + export_derive
             + forbid_unsafe
             + logos-derive
             + std
             - debug
             - state_machine_codegen
```

Whenever you're using a new library, you should check out the [docs](https://docs.rs/logos/0.16.1/logos/). And the docs for the [derive macro](https://docs.rs/logos/0.16.1/logos/derive.Logos.html) are... basically missing. If we go back to the top level of the crate there's a link to a [Logos Handbook](https://logos.maciej.codes/), but really that should be in the rustdoc.

In their handbook, there's a TOC entry for [`#[logos]`](https://logos.maciej.codes/attributes/logos.html) attribute usage. We'll want a `skip` entry for whitespace since Yagbas doesn't care about whitespace, and also a custom error type. The custom error enum has to implement `PartialEq` and `Default`. The default error value can, I guess (?), be generated by Logos if it can't match any TokenKind rule to what's in the source file. We'll also want some other errors of our own in a moment.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Logos)]
#[logos(skip r#"[[:space:]]"#)]
#[logos(error = TokenizerError)]
pub enum TokenKind {
  // TODO
}

/// This type should never be deliberately used outside of the `tokenizer`
/// module, but must be public because it appears in an interface.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum TokenizerError {
  #[default]
  LexerUnknown,
}
```

All of our regex values, including the `[[:space:]]` regex above, are going to be given as raw strings. This makes using `\` characters in the regex itself a lot simpler. Even if a particular string doesn't need any `\` at all, we just always use raw strings in this part of the code for consistency.

PS: I hope you know about regex basics, because we're about to use a lot of them.

Also, if you're using a language that *does* care about whitespace (eg: Python or Haskell) you'd leave off the `skip` attribute and then your parser would have to gracefully handle spacing tokens all through the token stream.

Now we start specifying `TokenKind` variants. First we'll write down all the stuff that makes a "token tree". These are the markers that group tokens together, and the groups can be nested within each other, but they have to balance out or problems happen.

```rust
  /* Token Tree Markers */
  #[regex(r"\[")]
  OpBracket,
  #[regex(r"\]")]
  ClBracket,
  #[regex(r"\{")]
  OpBrace,
  #[regex(r"\}")]
  ClBrace,
  #[regex(r"\(")]
  OpParen,
  #[regex(r"\)")]
  ClParen,
  #[regex(r"/\*")]
  OpCommentBlock,
  #[regex(r"\*/")]
  ClCommentBlock,
```

Having `()` and `[]` and `{}` groups is fairly expected, but *also* if we want to have block-style comments that can nest then we need to specify `/*` and `*/` as token kinds.

If your language uses `<>` to make token tree groups, then you might want to include them here with names like `OpAngle` and `ClAngle`. Since Yagbas works "like rust" most of the time, the `>` and `<` characters will often appear in math expressions "on their own", rather than being used for general token grouping. Rust only requires that `<>` balance with each other when it's specifically looking for a type expression, not all the time, so our `TokenKind` will think of them being more like just any other punctuation. It's all in how tyou think about it.

Now we'll include the "Varying" kinds of token: identifiers, number and string literals, and also comments. The precise rules are up to your language, but once again Yagbas tries to "be like rust" and so it should be pretty familiar.

* An `Ident` is a "C-style" ident where you can use letters or numbers or underscore, but you can't *start* with a number.
* A `LitNum` is a "rust-style" number where you start with a digit and then you can put any other word character, which allows things like `0_u8` to be a single valid literal number. In Yagbas, we also have a case to allow `%` as a binary literal prefix and `$` as a hexadecimal literal prefix. Actually parsing the content of a literal number isn't for the tokenizer to do, so we keep the regex simple and if someone writes `7wolverine` we can tell them that's wrong later in the compiler.
* A `LitStr` is double quotes around some text, and you can use `\"` to put a double quote within the string itself. On this type of string literal the compiler should interpret the various escape sequences like `\n` and `\t`.
* A `LitRawStr` is `r` then 0 or more `#` then `"` then anything until we end with `"` and the same number of `#`, and the stuff in the middle *doesn't* do escape sequences when the compiler interprets it. 
* The comment markers for doc comments, inner doc comments, and line comments are exactly what you'd expect. Since these go "until the end of the line" their "greedy" matcher has to be explicitly allowed with a logos flag. Not entirely sure why they designed it that way but whatever, we set the flag.

```rust
  /* Stuff Where You Slice The Source To Check What It Was */
  #[regex(r"[_a-zA-Z][_a-zA-Z0-9]*")]
  Ident,
  #[regex(r"((\$|%)[[:word:]]+|[[:digit:]][[:word:]]*)")]
  LitNum,
  #[regex(r#"""#, end_lit_string)]
  LitStr,
  #[regex(r#"r#*\""#, end_lit_raw_string)]
  LitRawStr,
  #[regex(r"///[^\r\n]*", allow_greedy = true)]
  CommentDoc,
  #[regex(r"//![^\r\n]*", allow_greedy = true)]
  CommentInnerDoc,
  #[regex(r"//[^\r\n]*", allow_greedy = true)]
  CommentLine,
```

We'll be using [logos callbacks](https://logos.maciej.codes/callbacks.html) for the literal string and literal raw string ending logic. Literal raw strings cannot be defined in terms of just a regex, because regex doesn't allow for the `#` count matching logic. If we also use a callback for standard literal strings then we can give a more specific error than the default error when there's no closing `"`.

First let's write the logic for ending a normal string literal. If we look at the callbacks page, we need to *accept* a mutable reference to our `Lexer`, and then we need to return... one of several potential return types. In this case, we want a successful lex to be `Ok(TokenKind::LitStr)`, which on the chart is means we look for the stuff that says `Ok(Token::Unit)`. On and error, we want to specify what error value it is. Confusingly, this means that our return type should *not* be `Result<TokenKind, TokenizerError>`, because that would try to pass the `Ok` value from our callback to `TokenKind::LitStr`, which doesn't take a value anyway. Instead, we need to make our callback return `Result<(), TokenizerError>`, then `logos` will convert that `()` into a `TokenKind::LitStr` internally.

This is not how I would have designed the API, to say the least. But that's it's how it works in `logos`.

```rust
fn end_lit_string(lex: &mut Lexer<TokenKind>) -> Result<(), TokenizerError> {
  todo!()
}
```

Alright, so what's the world state when the callback is called? The docs don't quite say! If we look in their example code though we can figure it out. The lexer will be pointed to the regex that matched. For a `LitStr`, that's going to be a single `"`. Let's just debug assert that. Never really hurts to have too many debug asserts.

```rust
fn end_lit_string(lex: &mut Lexer<TokenKind>) -> Result<(), TokenizerError> {
  let prefix = lex.slice();
  debug_assert!(prefix.len() == 1);
  debug_assert!(prefix == "\"");
  todo!()
}
```

Okay so the lexer is *pointed to* the opening double quote, now we need to find the ending double quote. When we do, we call `bump` to push the lexer up to 1 past that position (because the closing double quote is 1 byte itself).

```rust
fn end_lit_string(lex: &mut Lexer<TokenKind>) -> Result<(), TokenizerError> {
  let prefix = lex.slice();
  debug_assert!(prefix.len() == 1);
  debug_assert!(prefix == "\"");
  let remainder = lex.remainder();
  match remainder.find("\"") {
    Some(position) => {
      lex.bump(position + 1);
      Ok(())
    }
    None => {
      lex.bump(remainder.len());
      Err(TokenizerError::LitStrNoCloseQuote)
    }
  }
}
```

Let's write a `tokenize` function to make this lexer easily go:

```rust
pub fn tokenize(source: &str) -> impl Iterator<Item = Token> + Clone + '_ {
  TokenKind::lexer(source).spanned().map(|(res, range)| Token {
    kind: match res {
      Ok(kind) => kind,
      Err(TokenizerError::LexerUnknown) => TokenKind::LexerUnknown,
      Err(TokenizerError::LitStrNoCloseQuote) => TokenKind::LitStrNoCloseQuote,
    },
    span: Span { start: range.start, end: range.end },
  })
}
```

Now a test for all of this:

```rust
#[test]
fn test_tokenize() {
  let mut v: Vec<Token> = tokenize(".").collect();
  assert_eq!(v[0].kind, TokenKind::Period);

  v = tokenize("\"abc\"").collect();
  assert_eq!(v[0].kind, TokenKind::LitStr);
  assert_eq!(v[0].span.as_range(), 0..5);

  v = tokenize("\"").collect();
  assert_eq!(v[0].kind, TokenKind::LitStrNoCloseQuote);
}
```

And those tests pass, so our code is totally bug free. No bugs if the tests pass!

I don't like that magical seeming `5` in the test though. Constants without explanation are a bit of a code smell.

```rust
  let x = "\"abc\"";
  v = tokenize(x).collect();
  assert_eq!(v[0].kind, TokenKind::LitStr);
  assert_eq!(v[0].span.as_range(), 0..x.len());
```

Okay, so what we were really saying is that the length of the lexed token is the entire input string, since we're getting just one token out.

What happens if we add a tests case where we use an escaped `"` inside of our source.

```rust
  let x = "\"a\\\"bc\"";
  v = tokenize(x).collect();
  assert_eq!(v[0].kind, TokenKind::LitStr);
  assert_eq!(v[0].span.as_range(), 0..x.len());
```

Oh hek now the tests fail. The token that comes out is only 4 bytes long, not 7. It didn't interpret the backslashes correctly. Our callback has to be smarter. Every time it finds a closing `"`, it needs to see if that's *preceeded* by some backslashes, and ignore it if so.
