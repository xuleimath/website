---
slug: /
displayed_sidebar: GettingStartedSidebar
---
# Getting Started

## Just Starting Out

If you're new to the RISC Zero zkVM, these examples and explanations will get you oriented. Below is a tutorial that'll help you build our version of "Hello, World!" and an introduction to the default RISC Zero project template.

* [**Hello, Multiply!**](examples/rust_starter_tutorial.md) - Think of this as "Hello, World!" for the RISC Zero zkVM. By following this tutorial, you'll create a program that demonstrates a number is composite (and that you know its factors). If you're just getting started writing code for the zkVM, we recommend starting here.

* [**Understanding the Project Template**](examples/understanding_template.md) - Here we'll take a closer look at the default template for a RISC Zero zkVM project. You'll understand which parts are necessary and which can be changed, and get a feel for how the project components work together. (And if you like understanding the structure of a project before diving in, feel free to start here instead!)

## Project Examples

For more ideas about what's possible with RISC Zero, take a look at these examples, which feature explanations and tutorials for code available in our GitHub repositories:

* [**RISC Zero Battleship**](examples/battleship_rust.md) - To see a more complex use of the RISC Zero zkVM in action, take a look at this working Battleship game. Here, players use private game board states to track whether opponents have sunk their battleships.

* [**RISC Zero Digital Signatures**](examples/digital_signatures.md) - In this example, you'll see how to verifiably sign code run inside the RISC Zero zkVM. This example features a post-quantum digital signature generated using only SHA-2 as a cryptographic primitive.

* [**RISC Zero Password Validity Checker**](examples/password_checker.md) - In this example, you'll see Alice convince Bob's Identity Service that her password meets Bob's validity requirements. This example makes use of public shared outputs that Alice can write to the RISC Zero zkVM's `journal`.

# Open Source Repositories

* [**Rust Crates**](https://github.com/risc0/risc0#rust-crates) - If you're a Rust user, you'll find RISC Zero crates here, ready to be included in your existing projects.

* [**Contribute to RISC Zero**](http://www.github.com/risc0/risc0) - If you're interested in how RISC Zero projects for the zkVM work, or curious about contributing to this project, come take a look at our main project repository.
