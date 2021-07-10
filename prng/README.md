# A Little Bit About PRNG Stuff

Let's talk about [Pseudorandom Number Generators][wp-prng], or PRNGs for short.

Pseudorandom number generators are a deterministic way to take some state and perform a step so that you get a new state and also an output value.

In Rust we might have a general outline like this:
```rust=
type State = (); // TODO: pick a state type
type Output = (); // TODO: pick an output type

pub const fn example_generator(state: State) -> (State, Output) {
    // TODO: implement example_generator
    ((), ())
}
```

Usually we would make a struct to hold the generator state, and make the stepping function be a method on that struct, and so on and so forth. Today we'll just have a lot of functions, that's fine.

### Special Terms



### These Generators Are Not Cryptographically Secure

TODO

### I Hope Your Hardware Can Multiply

TODO

## Linear Congruential Generators

TODO

### Basic Theory

TODO

### Selecting Our Parameters

TODO

### Jumping The Generator

TODO

### Multiple Streams

TODO

### Extending The Period

TODO

## Permuted Congruential Generators

TODO

## Generating Bounded Integers

TODO

## Generating Normalized Floats

TODO

## Seeding Your Generator

TODO

<!-- Links -->

[wp-prng]: https://en.wikipedia.org/wiki/Pseudorandom_number_generator
[wp-lcg]: https://en.wikipedia.org/wiki/Linear_congruential_generator
[wp-pcg]: https://en.wikipedia.org/wiki/Permuted_congruential_generator
[wp-xorshift]: https://en.wikipedia.org/wiki/Xorshift
