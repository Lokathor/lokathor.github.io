
# Lokathor's Ecosystem Guide

Here's a guide to the stuff that I, Lokathor, personally use,
and that I think is high enough quality that you might also find it useful.

## Useful Libraries

* [![docs.rs](https://docs.rs/bytemuck/badge.svg) bytemuck](https://docs.rs/bytemuck/):
  A crate that lets you cast around between data types much more easily and safely than normal.
  The primary operations are that you can cast "owned" values (`T` -> `U`),
  by reference (`&T` -> `&U` and `&mut T` -> `&mut U`),
  and by slice (`&[T]` -> `&[U]` and `&mut [T]` -> `&mut [U]`). There's also a few helper functions
  for some of the common other casts you might want to do, such as `&mut T` -> `&mut [u8]`. It can even
  cast `Box` and `Vec` with a feature enabled! Neat stuff.

* [![docs.rs](https://docs.rs/wide/badge.svg) wide](https://docs.rs/wide/):
  Has an `f32x4` data type which closely models the `f32` type, but does all the operations in SIMD.
  It doesn't support _just_ the actual SIMD intrinsics, it also supports some of the math you'd build
  on top of those intrinsics, such as sine and cosine. The intent is that this can be a fairly simple
  drop-in replacement for the normal `f32` type when you have batches of values to process.

## Useful Binaries

* [![crates.io](https://img.shields.io/crates/v/gbafix.svg) gbafix](https://crates.io/crates/gbafix):
  Based on the C utility of the same name. This is simply a Rust version.

* [![crates.io](https://img.shields.io/crates/v/uwu.svg) uwu](https://crates.io/crates/uwu):
  The most important and critical piece of software that has ever graced this planet.
