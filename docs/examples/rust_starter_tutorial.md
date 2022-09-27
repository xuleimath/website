---
title: "Rust zkVM Tutorial: Writing a Prover"
---

Welcome to the RISC Zero zero-knowledge virtual machine (zkVM) Rust tutorial! Here, we introduce a simple Rust code project; we encourage you to modify and expand it to learn how the zkVM works and to get a head start on building your own programs. Reading this document should help you get acquainted with writing code for the zkVM. We recommend reading the following explanation in conjunction with the [RISC Zero Rust starter example](https://github.com/risc0/risc0-rust-starter) repository.

This tutorial will introduce you to working with the RISC Zero zkVM in Rust, including:
* How to write RISC Zero zkVM guest code using the `risc0-zkvm-guest` crate
* How to build RISC Zero zkVM guest code using the `risc0-zkvm-build` crate
* How to execute the RISC Zero zkVM from a host using the `risc0-zkvm` crate
* How and when to privately share data between the host and guest
* How and when to publish data from the guest to the public journal
* What artifacts are produced by the RISC Zero zkVM, and how to use those artifacts to attest to the execution of the guest code

This tutorial will _not_ include:
* The cryptographic theory behind the RISC Zero zkVM (see TODO REFERENCE)
* The internal components of the RISC Zero zkVM (see TODO REFERENCE)
* (TODO: Do we include this one?) Instructions for constructing a zero-knowledge proof from a raw ELF file, or the requirements for such a file (see TODO REFERENCE)
* Design considerations for programs that use multiple RISC Zero zkVM guest methods as part of larger systems to accomplish complex tasks (see TODO REFERENCE PROBABLY BATTLESHIP OR VOTING MACHINE)
* A detailed example of writing a verifier that can check whether a purported proof from an untrusted source is valid (but some comments on this topic will be provided)

## Structure of a RISC Zero zkVM Program
Like other virtual machines, the RISC Zero zkVM has both _host_ and _guest_ components. The guest component contains the code to be proven. The host component provides any required data to the guest, executes the guest code, and handles the guest's output.

In typical use cases, a RISC Zero zkVM program will actually be structured with _three_ components:
* Source code for the _guest_,
* Code that _builds_ the guest's source code into executable methods, and
* Source code for the _host_, which will call these built methods.

Each of these components uses its own associated RISC Zero crate:
* The _guest_ code uses the [`risc0-zkvm-guest` crate](https://docs.rs/risc0-zkvm-guest/latest)
* The _build_ code for building guest methods uses the [`risc0-build` crate](https://docs.rs/risc0-build/latest)
* The _host_ code uses the [`risc0-zkvm` crate](https://docs.rs/risc0-zkvm/latest)

It is possible to organize the files for these components in various ways. However, in code published by RISC Zero we use a standard directory structure for zkVM code, and we recommend you use this structure as well. See below for a diagram of this directory structure with annotations. You can also see this structure in the [Rust starter example repository](https://github.com/risc0/risc0-rust-starter).

```
project_name
├── Cargo.toml
├── methods
│   ├── Cargo.toml
│   ├── build.rs                           <-- Build (embed) code goes here
│   ├── guest
│   │   ├── Cargo.toml
│   │   ├── build.rs                       <-- Build (link) code goes here
│   │   └── src
│   │       └── bin
│   │           └── guest_method_name.rs   <-- Guest code goes here
│   └── src
│       └── lib.rs                         <-- Build (include) code goes here
└── project_or_component_name
    ├── Cargo.toml
    └── src
        └── main.rs                        <-- Host code goes here
```

Now let's go through writing code for each of these three components in detail.

## Guest code
First let's look at the how to write guest code. This is the code you will be proving you faithfully executed.

### Guest code: No-op
In the simplest case, the guest code can do nothing at all, as shown below:
```
#![no_std]
#![no_main]

risc0_zkvm_guest::entry!(main);

pub fn main() {
    // Do nothing
}
```

Let's see what each of these lines does.
```
#![no_std]
```
The guest code should be as lightweight as possible for performance reasons. So, since we aren't using std, we exclude it.
```
#![no_main]

risc0_zkvm_guest::entry!(main);
```
The guest code is never launched as a standalone Rust executable, so we specify `#![no_main]`. However, we must make the guest code available for the host to launch, and to do that we must specify which function to call when the host starts executing this guest code. We use the `risc0_zkvm_guest::entry!` macro to indicate the initial guest function to call, which in this case is `main`.
```
pub fn main() {
    // Do nothing
}
```
Here is the actual guest code. Notice that the function is named `main`, matching the name specified in `entry!`, so this is the function that will be called when the host launches the guest. In real use cases, you would do more than nothing in this function.

## Building Guest Methods

The `risc0-build` crate has two functions, [`embed_methods`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods.html) and [`link`](https://docs.rs/risc0-build/latest/risc0_build/fn.link.html), which are used to build guest code into a method (or methods) that the host can call. Simple use cases have no need to do any customization for the build step, and you can just call these functions as described below. For more complex cases, it is sometimes useful to replace `embed_methods` with [`embed_methods_with_options`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods_with_options.html) (see TODO FAQ ABOUT TOO MANY CYCLES for an example where you might want to specify embedding options).


### Linking
To link the guest code, simply add a `build.rs` file to the root of your `guest` directory containing the following:
```
fn main() {
    risc0_build::link();
}
```

### Embedding
To embed the guest methods, add a `build.rs` file to the methods directory where you want the methods embedded. This is where your host code will need to look to find the guest methods. A basic `build.rs` file for embedding methods looks as follows:
```
fn main() {
    risc0_build::embed_methods();
}
```
For more advanced cases, replace `embed_methods` with a call to [`embed_methods_with_options`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods_with_options.html) and set appropriate options for your use case.

### Including
Linking and embedding the guest methods using these [build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) creates source files in the Rust output directory. To make this code available to the host, you need to include these generated files somewhere the host can find them. So, in your methods directory, create a file `src/lib.rs` with the following include command:
```
include!(concat!(env!("OUT_DIR"), "/methods.rs"));
```

### Build dependencies in Cargo
Both linking and embedding depend on `risc0-build`. Since these happen in built scripts, Cargo needs to know they are _build_ dependencies. Therefore, in both the guest directory Cargo file (for linking) and the methods directory cargo file (for embedding), you need to include
```
[build-dependencies]
risc0-build = "0.11"
```
(or adjust the version number if you want to use a different version of risc0).

Additionally, the `embed_methods` code needs to know where to find the guest code. This is indicated with custom risc0 metadata in the methods directory cargo file, which looks something like
```
[package.metadata.risc0]
methods = ["guest"]
```
Here `"guest"` is the relative path to the root of the directory with the guest source code. This is the conventional location for the guest source code, so you don't need to change it if you are following the directory structure outlined above.

## Host code

Now let's look at the host code need to execute the guest.

### Host code: No-op

Let's look at the simplest case where the host doesn't need to communicate with the guest, first in full and then line by line:
```
use methods::{TODO_ID, TODO_PATH};
use risc0_zkvm::host::Prover;

fn main() {
    let mut prover = Prover::new(&std::fs::read(TODO_PATH).unwrap(), TODO_ID).unwrap();

    let receipt = prover.run().unwrap();

    receipt.verify(TODO_ID).unwrap();
}
```
We start with use declarations
```
use methods::{TODO_ID, TODO_PATH};
use risc0_zkvm::host::Prover;
```
For `Prover` this is straightforward, but the `methods` are coming from computer generated code. Specifically, the `methods.rs` [file you included earlier](#including) contains generated constants needed to call guest methods. For each [guest code file](#guest-code), two constants are generated: `<FILENAME>_ID` and `<FILENAME>_PATH` (where `<FILENAME>` is the name of the file rendered in all capital letters). The `<FILENAME>_ID` is a _method ID_, a cryptographic hash that will be committed to the receipt and allows you to convince a verifier that the code you proved is the same code you are showing to them. The `<FILENAME>_PATH` is a path to where your method was built.
```
fn main() {
```
The host is executed directly, so this is the normal Rust `main` function.
```
    let mut prover = Prover::new(&std::fs::read(TODO_PATH).unwrap(), TODO_ID).unwrap();
```
This creates a prover, which can be run to execute its associated guest code and produce a receipt proving execution. It must be initialized with the contents of an ELF file of the code to be executed and with a method ID. These have be created in the build step, and can be accessed via the `<FILENAME>_PATH` and `<FILENAME>_ID` constants.
```
    let receipt = prover.run().unwrap();
```
This line actually runs the guest code inside the prover, the result of which is a receipt proving the execution. From here we can transfer the receipt to anyone we wish to verify our code -- for the sake of this tutorial, we will do so in the same process.
```
    receipt.verify(TODO_ID).unwrap();
```
This line verifies that a receipt corresponds to the execution of guest code whose method ID is `<FILENAME>_ID`. It's not necessary for the prover to run this line to make a valid proof. Instead, this is needed by anyone who wishes to verify that they have an honest receipt.
