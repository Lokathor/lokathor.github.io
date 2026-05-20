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

We'll need the `TokenKind` and `Span` types too of course:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Logos)]
#[logos(skip r#"[[:space:]]"#)] // ignore whitespace between tokens
pub enum TokenKind {
  // TODO!!!!
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

We're going to derive the `Logos` trait on `TokenKind`. Setting this up correctly has many steps. Each step is small and simple, but there's still a lot of them.

On the `TokenKind` type itself we put a `logos` attribute and tell it that logos should skip over a particular regex. It's possible to pass non-regex info to logos, but for consistency we'll always give a regex. We're also passing a raw string here just for consistency, because lower down we'll have to use lots of `\` to escape stuff in the regexes and so those parts will look much better when we're using raw strings. The `[[:space:]]` text itself is a "regex class", and that means that anything that the regex engine calls a space we'll be fine with being whitespace in our language. It covers actual spaces of course, but also tabs, and newlines, and carriage returns, and probably more stuff like "Form feed" or whatever.

If you're working on a language that *does* care about white space sometimes, such as Python or Haskell, you wouldn't do this step, and then you'd get whitespace tokens in the tokenizer output, which you'd have the necessary control over in your parsing.

Now we start specifying all our kinds of tokens. First we'll write down all the stuff that makes a "token tree". These are the markers that group tokens together, and they can be nested into each other, but they have to appear in balance with each other. Having `()` and `[]` and `{}` groups is expected, but *also* if we want to have the usualy block-style comments then we need to specify `/*` and `*/` as tokens of their own. Filtering away all the comments is a step later on, right now we're just designating the tokens.

If your language uses `<>` to make token tree groups, then you might want to include them here with names like `OpAngle` and `ClAngle`. Since Yagbas works "like rust" most of the time, the `>` and `<` characters will often appear in math expressions rather than being used for general token grouping. Rust only requires that `<>` balance with each other when it's specifically looking for a type expression, not all the time.

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

Now we'll include the "Varying" kinds of token: identifiers, number and string literals, and also comments. The precise rules are up to your language, and logos will even tell you with a compile error if you have more than one kind that could overlap. If you're familiar with regex and familiar with Rust then these definitions should be pretty familiar.

* An Ident is a "C-style" ident where you can use letters or numbers or underscore, but you can't *start* with a number.
* A LitNum is a "rust-style" number where you start with a digit and then you can put any other word character, which allows things like `0_u8` to be a single valid literal number. In Yagbas, we also have a case to allow `%` as a binary literal prefix and `$` as a hexadecimal literal prefix. Actually parsing the content of a literal number isn't for the tokenizer to do, so we keep the regex simple and if someone writes `7wolverine` we can tell them that's wrong later in the compiler.
* A LitStr is exactly what you'd expect.
* A LitRawStr just specifies the regex to find the prefix to determine that we're on a LitRawStr, and then after that we put the name of a function which will determine where the end of the token goes. Specifying a function allows even more flexability than using a regex, but you have to write the function yourself, so we only do it when we have to. 
* The comment markers for doc comments, inner doc comments, and line comments are exactly what you'd expect. Since you could write a "greedy" matcher which checks all the way to the end of the input on accident, logos requires that you write `allow_greety=true` if you *really* wanted that.

```rust
  /* Stuff Where You Slice The Source To Check What It Was */
  #[regex(r"[_a-zA-Z][_a-zA-Z0-9]*")]
  Ident,
  #[regex(r"((\$|%)[[:word:]]+|[[:digit:]][[:word:]]*)")]
  LitNum,
  #[regex(r#""((\\")|[^"\\])*""#)]
  LitStr,
  #[regex(r#"r#*\""#, end_raw_string)]
  LitRawStr,
  #[regex(r"///[^\r\n]*", allow_greedy = true)]
  CommentDoc,
  #[regex(r"//![^\r\n]*", allow_greedy = true)]
  CommentInnerDoc,
  #[regex(r"//[^\r\n]*", allow_greedy = true)]
  CommentLine,
```

Now we put all the punctuation and keywords. There's a lot here, but it's all just "look for this fixed thing, and here's what to call it".

```rust
  /* Stuff That's Always One Thing */
  #[regex(r"bitbag")]
  KwBitbag,
  #[regex(r"break")]
  KwBreak,
  #[regex(r"const")]
  KwConst,
  #[regex(r"continue")]
  KwContinue,
  #[regex(r"else")]
  KwElse,
  #[regex(r"false")]
  KwFalse,
  #[regex(r"fn")]
  KwFn,
  #[regex(r"if")]
  KwIf,
  #[regex(r"let")]
  KwLet,
  #[regex(r"loop")]
  KwLoop,
  #[regex(r"mmio")]
  KwMmio,
  #[regex(r"ram")]
  KwRam,
  #[regex(r"return")]
  KwReturn,
  #[regex(r"rom")]
  KwRom,
  #[regex(r"static")]
  KwStatic,
  #[regex(r"struct")]
  KwStruct,
  #[regex(r"true")]
  KwTrue,

  #[regex(r"~")]
  Tilde,
  #[regex(r"`")]
  Backtick,
  #[regex(r"!")]
  Exclamation,
  #[regex(r"@")]
  AtSign,
  #[regex(r"#")]
  Hash,
  #[regex(r"\$")]
  Dollar,
  #[regex(r"%")]
  Percent,
  #[regex(r"\^")]
  Caret,
  #[regex(r"&")]
  Ampersand,
  #[regex(r"\*")]
  Asterisk,
  #[regex(r"-")]
  Minus,
  #[regex(r"\+")]
  Plus,
  #[regex(r"=")]
  Equal,
  #[regex(r"\|")]
  Pipe,
  #[regex(r"\\")]
  Backslash,
  #[regex(r":")]
  Colon,
  #[regex(r";")]
  Semicolon,
  #[regex(r"'")]
  Quote,
  #[regex(r"<")]
  LessThan,
  #[regex(r",")]
  Comma,
  #[regex(r">")]
  GreaterThan,
  #[regex(r"\.")]
  Period,
  #[regex(r"\?")]
  Question,
  #[regex(r"/")]
  Slash,
```

Finally we want some error token kinds for when things have gone wrong.

* One token kind will be for when things can't lex properly. This prevents us having to work with `Result<TokenKind, ()>` in a bunch of places. It's also possible to have other error types than just `()`, but we don't want to be using `Result` *at all*, so it doesn't really matter.
* The other will be an explicit token kind for the end of input. And any time we try to index forward in the token sequence and go out of bounds, that will also return the end of input token kind. Now we don't have to deal with `Option<TokenKind>` all over. This suggestion I didn't think of myself, it came from matklad's article.

```rust
  /// This is for when the file contains something, but the lexer doesn't know
  /// what to call it.
  LexerConfused,

  /// Virtual token used for when looking "out of bounds" of the token stream.
  ///
  /// This avoids the parser internals from needing `Option<TokenKind>`.
  EndOfFile,
```

And that's... All of our token kinds. Only a few dozen of them or so, not bad.

Now we need to add that function for finding the end of a raw string.

```rust

```