---
title: "Understanding the RISC Zero zkVM Starter Template"
---

The [RISC Zero Rust starter template](https://github.com/risc0/risc0-rust-starter) provides a starting point for RISC Zero zkVM projects. This article will describe what the template code does, and why we've written it this way. In particular, it should help you understand:
* The host, guest, and build components of RISC Zero zkVM programs
* How guest methods are built and made available to the host
* How the host calls guest methods
* What is included in Cargo files to be able to run the zkVM

This tutorial will _not_ include:
* How to create a project based on the starter template (see [Hello, Multiply](hello_multiply.md))
* The cryptographic theory behind the RISC Zero zkVM (see our [proof system explainers and reference materials](../explainers))
* The internal components of the RISC Zero zkVM (see our [Overview of the zkVM](../explainers/zkvm) article)
* Design considerations for programs that use multiple RISC Zero zkVM guest methods as part of larger systems to accomplish complex tasks (see our [Battleship example](battleship_rust.md))

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

It is possible to organize the files for these components in various ways. However, in code published by RISC Zero we use a standard directory structure for zkVM code, and we recommend you use this structure as well. See below for a diagram of this directory structure with annotations. You can also see this structure in the [Rust starter template repository](https://github.com/risc0/risc0-rust-starter).

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
│   │           └── method_name.rs         <-- Guest code goes here
│   └── src
│       └── lib.rs                         <-- Build (include) code goes here
└── project_or_component_name
    ├── Cargo.toml
    └── src
        └── main.rs                        <-- Host code goes here
```

Now let's go through these three components in detail.

## Guest code
The guest code is the code the prover wants to demonstrate is faithfully executed. The template starts from the simplest possible guest code -- its guest method does nothing:
```
#![no_std]
#![no_main]

risc0_zkvm_guest::entry!(main);

pub fn main() {
    // TODO: Implement your guest code here
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
    // TODO: Implement your guest code here
}
```
Here is the actual guest code. Notice that the function is named `main`, matching the name specified in `entry!`, so this is the function that will be called when the host launches the guest. In real use cases, you would do more than nothing in this function.

## Building Guest Methods

The `risc0-build` crate has two functions, [`embed_methods`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods.html) and [`link`](https://docs.rs/risc0-build/latest/risc0_build/fn.link.html), which are used to build guest code into a method (or methods) that the host can call. Simple use cases have no need to do any customization for the build step, and you can just call these functions as described below. For more complex cases, it is sometimes useful to replace `embed_methods` with [`embed_methods_with_options`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods_with_options.html) (see the [FAQ](../faq.md#zkp-system) for an example where you might want to specify embedding options).

These functions are called at build time using [Cargo build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html). The resulting files with the built methods must then be included so that the host can depend on them.

### Linking
The guest code is linked via a `build.rs` file in the root of the `guest` directory containing the following:
```
fn main() {
    risc0_build::link();
}
```

### Embedding
The guest methods are embedded using a `build.rs` file in the methods directory where you want the methods embedded. This is where the host code will need to look to find the guest methods. A basic `build.rs` file for embedding methods looks as follows:
```
fn main() {
    risc0_build::embed_methods();
}
```
For more advanced cases, replace `embed_methods` with a call to [`embed_methods_with_options`](https://docs.rs/risc0-build/latest/risc0_build/fn.embed_methods_with_options.html) and set appropriate options for your use case.

### Including
Linking and embedding the guest methods using these [build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) creates source files in the Rust output directory. To make this code available to the host, these generated files must be included somewhere the host can find them. So the methods directory contains a file `src/lib.rs` with the following include command:
```
include!(concat!(env!("OUT_DIR"), "/methods.rs"));
```

### Build dependencies in Cargo
Both linking and embedding depend on `risc0-build`. Since these happen in built scripts, Cargo needs to know they are _build_ dependencies. Therefore, in both the guest directory Cargo file (for linking) and the methods directory cargo file (for embedding), we include
```
[build-dependencies]
risc0-build = "0.11"
```
(or adjust the version number if you want to use a different version of risc0).

Additionally, the `embed_methods` code needs to know where to find the guest code. This is indicated with custom `risc0` metadata in the methods directory cargo file, which looks like
```
[package.metadata.risc0]
methods = ["guest"]
```
Here `"guest"` is the relative path to the root of the directory with the guest source code, and can be adjusted if you aren't following the directory structure outlined above.

## Host code

Now let's look at the host code need to execute the guest. The code in the template does not communicate with the guest or provide a method for sending the receipt to an external verifier. Let's look at the code first in full, then line by line:
```
use methods::{METHOD_NAME_ID, METHOD_NAME_PATH};
use risc0_zkvm::host::Prover;

fn main() {
    // Make the prover.
    let method_code = std::fs::read(METHOD_NAME_PATH)
        .expect("Method code should be present at the specified path; did you use the correct *_PATH constant?");
    let mut prover = Prover::new(&method_code, METHOD_NAME_ID)
        .expect("Prover should be constructed from valid method source code and corresponding method ID");

    // Run prover & generate receipt
    let receipt = prover.run()
        .expect("Valid code should be provable if it doesn't overflow the cycle limit. See `embed_methods_with_options` for information on adjusting maximum cycle count.");

    // Optional: Verify receipt to confirm that recipients will also be able to verify your receipt
    receipt.verify(METHOD_NAME_ID)
        .expect("Code you have proven should successfully verify; did you specify the correct method ID?");
}

```
We start with use declarations
```
use methods::{METHOD_NAME_ID, METHOD_NAME_PATH};
use risc0_zkvm::host::Prover;
```
For `Prover` this is straightforward, but the `methods` are coming from computer generated code. Specifically, the `methods.rs` [file you included earlier](#including) contains generated constants needed to call guest methods. For each [guest code file](#guest-code), two constants are generated: `<FILENAME>_ID` and `<FILENAME>_PATH` (where `<FILENAME>` is the name of the file rendered in all capital letters). The `<FILENAME>_ID` is a _method ID_, a cryptographic hash that will be committed to the receipt and allows you to convince a verifier that the code you proved is the same code you are showing to them. The `<FILENAME>_PATH` is a path to where your method was built.
```
fn main() {
```
The host is executed directly, so this is the normal Rust `main` function.

We will replace `expect`s with `unwrap`s in the following lines so we can focus on the core functionality:
```
    let method_code = std::fs::read(METHOD_NAME_PATH).unwrap();
    let mut prover = Prover::new(&method_code, METHOD_NAME_ID).unwrap();
```
This creates a prover, which can be run to execute its associated guest code and produce a receipt proving execution. It must be initialized with the contents of an ELF file of the code to be executed and with a method ID. These have be created in the build step, and can be accessed via the `<FILENAME>_PATH` and `<FILENAME>_ID` constants.
```
    let receipt = prover.run().unwrap();
```
This line actually runs the guest code inside the prover, the result of which is a receipt proving the execution. From here we can transfer the receipt to anyone we wish to verify our code -- in the template, we do so in the same process for simpilicity.
```
    receipt.verify(METHOD_NAME_ID).unwrap();
```
This line verifies that a receipt corresponds to the execution of guest code whose method ID is `<FILENAME>_ID`. It's not necessary for the prover to run this line to make a valid proof. Instead, this is needed by anyone who wishes to verify that they have an honest receipt.
