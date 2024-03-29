# Storq design

As a refresher, Storq is an item store that specialises in ordered queues for
work processors. Each queue can have multiple orderings applied, like views on
a table, and orderings can be completely arbitrary: each ordering is computed
via a keying function provided by the application and stored within Storq.

Storq queues, orderings (indexes), and keying functions are named by the
application and can be renamed, so long as they remain unique in their kind.
Like sled, Storq supports arbitrary byte-strings for both names and items.

## A construction on top of sled

Storq is built on top of sled.

sled is an **advanced** key-value store. There are three features that
distinguish sled from most other key-value stores and that Storq relies upon
absolutely:

1. Multiple namespaces per database. It's not one large key-value space, it's
   several distinct _trees_ within a single database.

2. Full set iteration, and equality, inequality, and range querying. Each
   namespace is a B-tree, so not only can sled support random-access key-value
   structures, unordered and ordered collections as well as indexes can also be
   built directly, without serialising a B-tree on top of raw key-value.

3. Atomic operations and ACID transactions. There is a good range of modern
   operations with excellent documentation for precisely defined semantics,
   including several flavours of CAS. Transactions can span several trees at
   once and support both reads and writes.

Additionally, sled has arbitrary byte-string keys and values, so anything can
be encoded in any way the application requires.

## Common considerations

 - Numbers are always encoded little-endian.

## Names: forward and backward

There are the needs to retrieve trees based on names, and to retrieve the name
of a particular tree. Both operations are supported by a single **name table**.
That name table is a tree identified by the byte for `n`.

Each kind of tree has a letter assigned:

 - Queues are **Q**,
 - Indexes are **I**,
 - Functions are **F**.

This is used as a prefix in the name table. Uppercase is for forward
translation (getting the tree given a name), lowercase for reverse translation
(getting the name of a tree). The letters are encoded (Unicode) as a byte.

Each tree has its own unique 64-bit ID.

 - Forward: To retrieve a tree ID for a name, construct a key consisting of the
   kind's letter in uppercase as a byte, followed by the name byte-string.
   Query the name table, then parse the value if present as a `u64`.

 - Backward: To retrieve a name given a tree ID, construct a key consisting of
   the kind's letter in lowercase as a byte, followed by the 8 bytes of the ID.
   Query the name table, then return the value if present.

## Queues: actually just big bags o stuff

Each queue is a separate tree. The tree identifier is the byte for `q` followed
by the 8 bytes for the queue's ID. Items are added under generated unique IDs.
The only special thing is that adding an item to a queue also updates all its
associated indexes.

## Indexes: the interesting part

Each index is a separate tree. The tree identifier has this layout:

```
|  u8 | --- u64 --- |    u8    | --- u64 --- |  u8  | --- u64 --- |
| `i` |      ID     | revision |   queue ID  | mode | function ID |
```

The revision is a mechanism to allow rebuilding an index online. Revision 0 is
the active one, and when an index is being rebuilt the temporary index's
revision is 1. The presence of more than one revision for an index indicates
that it is being rebuilt; this may in the future be used for multiple
concurrent rebuilds, or to be reinterpreted as a bitflags structure. This is
the only mutable portion of the identifier, and in a very limited sense.

The queue ID binds the index to a particular queue.

The mode works out as a C-style enum defined thus:

```rust
#[repr(u8)]
enum Mode {
  // 0 is reserved
  OrderedHash = 1,
}
```

 - `OrderedHash` is a hybrid solution where the keying function's output has a
   SipHash of the item appended, as a discriminant when the keying function
   returns identical keys.

Further modes may be added later.

The function ID binds a particular keying function to the index.

In a relational sense:

 - A queue has many indexes, and an index has one queue;
 - An index has one function, and a function has many indexes.

The priciple of operation of the index is extremely simple: compute the key,
then insert, delete, or retrieve the associated item's ID, or range-query the
index for multiple of the same.

The key is computed according to the mode, using the keying function and/or the
item's contents.

## Keying functions: arbitrary programs

"Function" is a bit of a misnomer: these are WebAssembly modules. The interface
is defined thus:

 - Imported function:
   `log(ptr: i32, len: i32)`.

 - Imported memory:
   `key_space`, at index 0.

 - Exported function:
   `key_factory(ptr: i32, len: i32) -> i32`.
 
 - Exported global:
   `KEY_LENGTH: u8`.

The `log` function is to be used for diagnostics and debug. Currently it prints
to screen, though that may change.

The `KEY_LENGTH` (immutable) global is described as `u8` but should be WASM
type `i32`, so long as its value fits in a `u8`.

As an alternative to the exported global, if you cannot get it working within
your source language, you may declare an exported function `key_length` (note
the lowercase) that returns the same.

It is passed (the location of) a byte buffer in the `key_space` memory, to be
read as the input. It should write its output to that same memory and return
its pointer. The module defines the length of the output itself as the
`KEY_LENGTH` global defined above, so the factory never needs to return that.
The input buffer may vary in length at every call.

The return value may be negative in case of errors. At this point no standard
errors have been defined, but pre-emptively the range 0 to –1024 is reserved.

The keying function is thus given an item's contents as argument and must
return the corresponding key.

Functions are stored in their compiled binary WASM form in the tree named `f`
under a generated ID.
