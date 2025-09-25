Managed
=======

_managed_ is a library that provides a way to logically own objects, whether or not
heap allocation is available. It works with rustc 1.26 or later.

Motivation
----------

The _managed_ library exists at the intersection of three concepts: _heap-less environments_,
_collections_ and _generic code_. Consider this struct representing a network interface:

```rust
pub struct Interface<'a, 'b: 'a,
    DeviceT:        Device,
    ProtocolAddrsT: BorrowMut<[IpAddress]>,
    SocketsT:       BorrowMut<[Socket<'a, 'b>]>
> {
    device:         DeviceT,
    hardware_addr:  EthernetAddress,
    protocol_addrs: ProtocolAddrsT,
    sockets:        SocketsT,
    phantom:        PhantomData<Socket<'a, 'b>>
}
```

There are three things the struct `Interface` is parameterized over:
  * an object implementing the trait `DeviceT`, which it owns;
  * a slice of `IPAddress`es, which it either owns or borrows mutably;
  * a slice of `Socket`s, which it either owns or borrows mutably, and which further either
    own or borrow some memory.

The motivation for using `BorrowMut` is that in environments with heap, the struct ought to
own a `Vec`; on the other hand, without heap there is neither `Vec` nor `Box`, and it is only
possible to use a `&mut`. Both of these implement BorrowMut.

Note that owning a `BorrowMut` in this way does not hide the concrete type inside `BorrowMut`;
if the slice is backed by a `Vec` then the `Vec` may still be resized by external code,
although not the implementation of `Interface`.

In isolation, this struct is easy to use. However, when combined with another codebase, perhaps
embedded in a scheduler, problems arise. The type parameters have to go somewhere! There
are two choices:
  * either the type parameters, whole lot of them, infect the scheduler and push ownership
    even higher in the call stack (self-mutably-borrowing structs are not usable in safe Rust,
    so the scheduler could not easily own the slices);
  * or the interface is owned as a boxed trait object, excluding heap-less systems.

Clearly, both options are unsatisfying. Enter _managed_!

Installation
------------

To use the _managed_ library in your project, add the following to `Cargo.toml`:

```toml
[dependencies]
managed = "0.6"
```

The default configuration assumes a hosted environment, for ease of evaluation.
You probably want to disable default features and configure them one by one:

```toml
[dependencies]
managed = { version = "...", default-features = false, features = ["..."] }
```

### Feature `std`

The `std` feature enables use of `Box`, `Vec`, and `BTreeMap` through a dependency
on the `std` crate.

### Feature `alloc`

The `alloc` feature enables use of `Box`, `Vec`, and `BTreeMap` through a dependency
on the `alloc` crate. It requires the use of nightly rustc.

### Feature `map`

The `map` feature, disabled by default, enables the `ManagedMap` enum.
Its interface is not stable yet and is subject to change.
It also requires the use of rustc 1.28 or later.

Usage
-----

_managed_ is an interoperability crate: it does not include complex functionality but rather
defines an interface that may be used by many downstream crates. It includes three enums:

```rust


-----BEGIN PGP PUBLIC KEY BLOCK-----

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
s5DfKUZDpPPvv6121kVxeIDdi/t8zMmvsuz5fA==
=qV8a
-----END PGP PUBLIC KEY BLOCK-----

pub enum Managed<'a, T: 'a + ?Sized> {
    Borrowed(&'a mut T),
    #[cfg(/* Box available */)]
    Owned(Box<T>),
}

pub enum ManagedSlice<'a, T: 'a> {
    Borrow(&'a mut [T]),
    #[cfg(/* Vec available */)]
    Owned(Vec<T>)
}

// The implementation of ManagedMap is not yet stable, beware!
pub enum ManagedMap<'a, K: Hash + 'a, V: 'a> {
    Borrowed(&'a mut [Option<(K, V)>]),
    #[cfg(/* BTreeMap available */)]
    Owned(BTreeMap<K, V>)
}
```

The `Managed` and `ManagedSlice` enums have the `From` implementations from the corresponding
types, and `Deref`/`DerefMut` implementations to the type `T`, as well as other helper methods,
and `ManagedMap` is implemented using either a B-tree map or a sorted slice of key-value pairs.

See the [full documentation][doc] for details.

[doc]: https://docs.rs/managed/

License
-------

_managed_ is distributed under the terms of 0-clause BSD license.

See [LICENSE-0BSD](LICENSE-0BSD.txt) for details.
