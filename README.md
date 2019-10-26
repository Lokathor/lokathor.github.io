
# Lokathor's Ecosystem Guide

Kinda sorted by "how portable" is the code, or maybe "how much setup does the code require" might be a good way to put it.

The first tier is "core only", and then each next tier is basically an extra thing you might need to handle in your setup.

## Core Only

### cfg-if

The [cfg-if](https://docs.rs/cfg-if) crate is the ultimate answer to conditional compilation. It takes in a macro expression formatted to look like an `if-else` expression chain, and spits out whichever block of the expression matches the compile time configuration. If you know how to use `if-else` then you already know how to use this crate. Each conditional test is a `#[cfg()]` conditional, it's as easy as that. The crate is just a single macro, so it works anywhere.

### static_assertions
### bytemuck
### voladdress

## Core + Alloc

## FFI / OS

### winapi
### fenestroj
### getrandom

## Rust Standard Library

### wide
