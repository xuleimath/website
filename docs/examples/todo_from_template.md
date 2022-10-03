TODO: Title, front matter

TODO: Explain that the code will be proving a number is composite by proving it is the output of multiplying two nontrivial factors

This tutorial will introduce you to working with the RISC Zero zkVM in Rust, including:
* How to write RISC Zero zkVM guest code using the `risc0-zkvm-guest` crate
* How to execute the RISC Zero zkVM from a host using the `risc0-zkvm` crate
* How to use the [starter template repository](https://github.com/risc0/risc0-rust-starter) to start a RISC Zero zkVM project starting from some prewritten boilerplate code
* How and when to privately share data between the host and guest
* How and when to publish data from the guest to the public journal
* What artifacts are produced by the RISC Zero zkVM, and how to use those artifacts to attest to the execution of the guest code

This tutorial will not include:
* TODO

## Creating a new project

Let's start by creating a new project named `my-factors` by cloning the [starter template](https://github.com/risc0/risc0-rust-starter):
```
git clone https://github.com/risc0/risc0-rust-starter my-factors
```
(If you intend to publish your project, you should instead follow the [directions for creating a repository from a template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template) on GitHub. If you are just following along with this example, then that is unnecessary complexity and you can just use `git clone` as described above.)

This will set up a basic RISC Rust zkVM project, which includes host code that runs a zkVM guest. You can run this code immediately if you like:
```
cd my-factors
cargo run
```
Since you haven't added any functionality yet, there won't be any output.

## Writing guest code

The guest code is the code that will be verified. In our case, we want verifiers to be convinced that we constructed the output number by multiplying two nontrivial factors, so we will implement multiplication in the guest.

The guest code is located in `methods/guest/src/bin/method_name.rs`. There are two `TODO` comments in the file that you'll need to implement. The first is to rename the file -- let's change the name from `method_name.rs` to `multiply.rs`, since the guest code will multiply two numbers.

The second `TODO` is to actually implement the guest code. The first thing we will need to do is read private data from the host, namely the factors to be multiplied.
