# Typetag

[![Build Status](https://api.travis-ci.org/dtolnay/typetag.svg?branch=master)](https://travis-ci.org/dtolnay/typetag)
[![Latest Version](https://img.shields.io/crates/v/typetag.svg)](https://crates.io/crates/typetag)
[![Rust Documentation](https://img.shields.io/badge/api-rustdoc-blue.svg)](https://docs.rs/typetag)

**Serde serializable and deserializable trait objects.**

This crate provides a macro for painless serialization of `&dyn Trait` trait
objects and serialization + deserialization of `Box<dyn Trait>` trait objects.

Let's dive into the example and I'll explain some more below.

```toml
[dependencies]
typetag = "0.0"
```

*Supports rustc 1.31+*

<br>

## Example

Suppose I have a trait `WebEvent` and I require that every implementation of the
trait be serializable and deserializable so that I can send them to my
ad-serving AI. Here are just the types and trait impls to start with:

```rust
trait WebEvent {
    fn inspect(&self);
}

#[derive(Serialize, Deserialize)]
struct PageLoad;

impl WebEvent for PageLoad {
    fn inspect(&self) {
        println!("200 milliseconds or bust");
    }
}

#[derive(Serialize, Deserialize)]
struct Click {
    x: i32,
    y: i32,
}

impl WebEvent for Click {
    fn inspect(&self) {
        println!("negative space between the ads: x={} y={}", self.x, self.y);
    }
}
```

We'll need to be able to send an arbitrary web event as JSON to the AI:

```rust
fn send_event_to_money_factory(event: &dyn WebEvent) -> Result<()> {
    let json = serde_json::to_string(event)?;
    somehow_send_json(json)?;
    Ok(())
}
```

and receive an arbitrary web event as JSON on the server side:

```rust
fn process_event_from_clickfarm(json: &str) -> Result<()> {
    let event: Box<dyn WebEvent> = serde_json::from_str(json)?;
    overanalyze(event)?;
    Ok(())
}
```

The introduction claimed that this would be painless but I'll let you be the
judge.

First stick an attribute on top of the trait.

```rust
#[typetag::serde(tag = "type")]
trait WebEvent {
    fn inspect(&self);
}
```

Then stick a similar attribute on all those impl blocks too.

```rust
#[typetag::serde]
impl WebEvent for PageLoad {
    fn inspect(&self) {
        println!("200 milliseconds or bust");
    }
}

#[typetag::serde]
impl WebEvent for Click {
    fn inspect(&self) {
        println!("negative space between the ads: x={} y={}", self.x, self.y);
    }
}
```

And now it works as described. All in all, three lines were added!

<br>

# What?

Trait objects are serialized by this library like Serde enums. Every impl of the
trait (anywhere in the program) looks like one variant of the enum.

All three of Serde's tagged [enum representations] are supported. The one shown
above is the "internally tagged" style so our two event types would be
represented in JSON as:

[enum representations]: https://serde.rs/enum-representations.html

```json
{"type":"PageLoad"}
{"type":"Click","x":10,"y":10}
```

The choice of enum representation is controlled by the attribute that goes on
the trait definition. Let's check out the "adjacently tagged" style:

```rust
#[typetag::serde(tag = "type", content = "value")]
trait WebEvent {
    fn inspect(&self);
}
```

```json
{"type":"PageLoad","value":null}
{"type":"Click","value":{"x":10,"y":10}}
```

and the "externally tagged" style, which is Serde's default for enums:

```rust
#[typetag::serde]
trait WebEvent {
    fn inspect(&self);
}
```

```json
{"PageLoad":null}
{"Click":{"x":10,"y":10}}
```

Separately, the value of the tag for a given trait impl may be defined as part
of the attribute that goes on the trait impl. By default the tag will be the
type name when no name is specified explicitly.

```rust
#[typetag::serde(name = "mouse_button_down")]
impl WebEvent for Click {
    fn inspect(&self) {
        println!("negative space between the ads: ({}, {})", self.x, self.y);
    }
}
```

<br>

## So many questions

- *Does it work if the trait impls are spread across different crates?* **Yes**

  Serialization and deserialization both support every single impl of the trait
  across the dependency graph of the final program binary.

- *Does it work in non-self-describing data formats like Bincode?* **Yes**

  All three choices of enum representation will round-trip correctly through
  compact binary formats including Bincode.

- *Does it support non-struct types?* **Yes**

  The implementations of the trait can be structs, enums, primitives, or
  anything else supported by Serde. The Serialize and Deserialize impls may be
  derived or handwritten.

- *Didn't someone explain to me why this wasn't possible?* **Yes**

  It might have been me.

<br>

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
</sub>
