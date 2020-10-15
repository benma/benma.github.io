---
layout: post
title: A pitfall of Rust's move/copy/drop semantics and zeroing data
---

We are using Rust extensively in the firmware of the [BitBox02](https://shiftcrypto.ch/bitbox02/)
hardware wallet. In a security device like this, you don't want to leave sensitive material in
memory for longer than necessary. In particular, when the value is being dropped, the memory should
be safely overwritten with zeroes, to mitigate the risks of the memory leaking.

[zeroize](https://docs.rs/zeroize/1.1.1/zeroize/) is a crate designed to make this task easy and
safe. Besides ensuring that the compiler does not optimize away the instructions, it allows you to
wrap your types in `zeroize::Zeroize<>`, so the value will be automatically zeroed on drop.

Coming from C, writing Rust feels a whole lot safer. So much in fact that I started to blindly trust
that things "just work" as expected. In this case, I expected that when I wrap my `TYPE` with
`zeroize::Zeroizing<TYPE>`, there would be no trace of the data after the value is dropped.

It turns out it is still quite easy to accidentally leave copies of the sensitive data in memory. Of
course, even Rust's powerful type system and high level of crates quality are not a silver
bullet. The zeroize docs even helpfully [describe those
pitfalls](https://docs.rs/zeroize/1.1.1/zeroize/#stackheap-zeroing-notes), but I still somehow
failed to grasp the meaning at first. [x1ddos](https://github.com/x1ddos) helpfully spend a few
hours with me digging into this.

Let's dive into an example:

```rust
#[derive(Debug)]
struct EncryptionKey([u8; 4]);

fn get_encryption_key() -> EncryptionKey {
    EncryptionKey(*b"AKey")
}

fn main() {
    let encryption_key = get_encryption_key();

    println!("Using key to encrypt stuff: {:?}", encryption_key);

    drop(encryption_key);

    println!("Rest of the program.");
}
```
Output:
```
Using key to encrypt stuff: EncryptionKey([65, 75, 101, 121])
Rest of the program.
```

By default, dropped values are not zeroed, so the value remains in memory until the same memory is
overwritten by another use:

```rust
#[derive(Debug)]
struct EncryptionKey([u8; 4]);

fn get_encryption_key() -> EncryptionKey {
    EncryptionKey(*b"AKey")
}

fn main() {
    let encryption_key = get_encryption_key();
    let ptr = encryption_key.0.as_ptr();

    println!("Using key to encrypt stuff: {:?}", encryption_key);

    println!("About to drop.");
    drop(encryption_key);
    println!("Dropped.");

    // Your output may vary, reading this memory is undefined behavior.
    println!("Memory: {:?}", unsafe {
        core::slice::from_raw_parts(ptr, 4)
    });
}
```
Output:
```
Using key to encrypt stuff: EncryptionKey([65, 75, 101, 121])
About to drop.
Dropped.
Memory: [65, 75, 101, 121]
```

As we can see, the key is still in memory, but the goal is to get rid of that toxic waste.

Let's try to use zeroize to fix this:

```rust
use zeroize::Zeroize;

#[derive(Debug)]
struct EncryptionKey([u8; 4]);

impl Drop for EncryptionKey {
    fn drop(&mut self) {
        self.0.zeroize();
        println!("Zeroed. Remaining value: {:?}", self.0);
    }
}

fn get_encryption_key() -> EncryptionKey {
    EncryptionKey(*b"AKey")
}

fn main() {
    let encryption_key = get_encryption_key();
    let ptr = encryption_key.0.as_ptr();

    println!("Using key to encrypt stuff: {:?}", encryption_key);

    println!("About to drop.");
    drop(encryption_key);
    println!("Dropped.");

    println!("Memory: {:?}", unsafe {
        core::slice::from_raw_parts(ptr, 4)
    });
}
```
Output:
```
Using key to encrypt stuff: EncryptionKey(Zeroizing([65, 75, 101, 121]))
About to drop.
Zeroed. Remaining value: Zeroizing([0, 0, 0, 0])
Dropped.
Memory: [65, 75, 101, 121]
```

Still, the memory is not properly wiped. What is going on? Let's inspect the actual pointers to
where the data is stored in RAM:

```rust
use zeroize::Zeroize;

#[derive(Debug)]
struct EncryptionKey([u8; 4]);

impl Drop for EncryptionKey {
    fn drop(&mut self) {
        println!("Pointer when zeroing: {:p}", self.0.as_ptr());
        self.0.zeroize();
        println!("Zeroed. Remaining value: {:?}", self.0);
    }
}

fn get_encryption_key() -> EncryptionKey {
    let key = EncryptionKey(*b"AKey");
    println!("Pointer at creation: {:p}", key.0.as_ptr());
    key
}

fn main() {
    let encryption_key = get_encryption_key();
    let ptr = encryption_key.0.as_ptr();

    println!("Pointer when using: {:p}", encryption_key.0.as_ptr());
    println!("Using key to encrypt stuff: {:?}", encryption_key);

    println!("About to drop.");
    drop(encryption_key);
    println!("Dropped.");

    println!("Memory: {:?}", unsafe {
        core::slice::from_raw_parts(ptr, 4)
    });
}

```
Output:
```
Pointer at creation: 0x7ffd632b0ba8
Pointer when using: 0x7ffd632b0c90
Using key to encrypt stuff: EncryptionKey([65, 75, 101, 121])
About to drop.
Pointer when zeroing: 0x7ffd632b0c10
Zeroed. Remaining value: [0, 0, 0, 0]
Dropped.
Memory: [65, 75, 101, 121]
```

Oh my, three different memory locations! How can this be?

The answer is that in Rust, **moving** a value compiles into a **memory copy** in the general
case. Sometimes, the compiler can be smart and optimize away a memory copy, but other times it is
impossible. For example, the `key` var in `get_encryption_key()` is a local stack variable, so
returning it (moving it out of the function) must be a memory copy under the hood.

What about the manual `drop()`? Same thing: the value is *moved* into the `drop()` function, but the
value is copied in memory while doing so.

In most cases, this semantics is exactly what you want as a programmer. The value is dropped only
once at the final location, and with a drop, the variable is moved into a sink and cannot be used
again after. The compiler would fail otherwise.

In this specific situation however, we would rather have the `Drop` implementation be called every
step of the way. This is of course not possible today. I hope there will be a compiler addition that
gives more fine grained control over memory copies.

To fix this, stack vars can be allocated at the top and pushed down into functions as mutable
arguments, but that leads to hard to understand and hard to maintain code. It is much easier to use
the heap from the start, where the location is permanent (*caveats apply here too, as described by
the zeroize docs!*).

What is copied is not the underlying data, but just the `Box` metadata:

```rust
use zeroize::Zeroize;

#[derive(Debug)]
struct EncryptionKey(Box<[u8; 4]>);

impl Drop for EncryptionKey {
    fn drop(&mut self) {
        println!("Pointer when zeroing: {:p}", self.0.as_ptr());
        self.0.zeroize();
        println!("Zeroed. Remaining value: {:?}", self.0);
    }
}

fn get_encryption_key() -> EncryptionKey {
    let key = EncryptionKey(Box::new(*b"AKey"));
    println!("Pointer at creation: {:p}", key.0.as_ptr());
    key
}

fn main() {
    let encryption_key = get_encryption_key();
    let ptr = encryption_key.0.as_ptr();

    println!("Pointer when using: {:p}", encryption_key.0.as_ptr());
    println!("Using key to encrypt stuff: {:?}", encryption_key);

    println!("About to drop.");
    drop(encryption_key);
    println!("Dropped.");

    println!("Memory: {:?}", unsafe {
        core::slice::from_raw_parts(ptr, 4)
    });
}
```
Output:
```
Pointer at creation: 0x558449695b40
Pointer when using: 0x558449695b40
Using key to encrypt stuff: EncryptionKey([65, 75, 101, 121])
About to drop.
Pointer when zeroing: 0x558449695b40
Zeroed. Remaining value: [0, 0, 0, 0]
Dropped.
Memory: [0, 0, 0, 0]
```

All clear!
