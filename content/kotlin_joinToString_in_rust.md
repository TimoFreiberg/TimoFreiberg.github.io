+++
title = "Porting a Kotlin API to Rust"
date = 2020-12-21
draft = true
+++

Intro:
Frequently used in Katas/Advent of Code or when aggregating error messages.

Stark difference between Kotlin and Rust:

```rust
errors
    .iter()
    .map(|e| e.to_string())
    .collect::<Vec<_>>()
    .join(", ");
```

```kotlin
errors.joinToString { it.toString() }
```

Changed Patterns:
* Use config struct for default values

Rust cons:
* No default arguments
* Have to think about borrowing vs owning

Kotlin cons:
* 
