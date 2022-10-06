--
title: "Hello, Multiply: Write Your First Program for the RISC Zero zkVM"
--

In this guide, we'll be writing a program that demonstrates a number is composite and *we know its factors*. If you'd like to start with an under-the-hood explanation of what this program will do, we've written one [here](understanding-hello-multiply.md). To jump ahead and see the finished product, look in [this repository](https://github.com/risc0/risc0-rust-examples/)If you're ready to code, let's dive in!

# Step 1: Clone the template repository

To get started, clone the [Rust starter template](https://github.com/risc0/risc0-rust-starter). (An explanation of the template code is [also available](rust_starter_tutorial.md).)

# Step 2: Modify file names (and their references)

## Name the new project

Let's give our "Hello, Multiply!" project a name. Rename the folder `project_or_component name` to `factors`.

Update the corresponding workspace member entry in the top-level `Cargo.toml` file:
```
[workspace]
members = [
    "methods",
    "factors"
]
```

## Give the guest program a name

This project will call a program that executes on the `guest zkVM`. It's currently named `methods/guest/src/bin/method_name.rs`. We want to name it something that represents what the guest program does -- let's call it `multiply.rs`.

In order to access this guest code from the host driver program, the host program `factors/src/main.rs` includes two guest methods:

```
use methods::{METHOD_NAME_ID, METHOD_NAME_PATH};
```
Both of these must be changed to reflect the new guest program name:
```
use methods::{MULTIPLY_ID, MULTIPLY_PATH};
```
(As an aside, if you add more than one callable guest program to your next RISC Zero zkVM project, you'll need to include these `ID` and `path` references once for each guest file.)

While we're at it, let's change the rest of the references in `factors/src/main.rs`. Don't worry about why these lines are included yet; for now, we're just being diligent not to leave dead references behind. Here are the three lines, updated with `MULTIPLY_PATH`, `MULTIPLY_ID`, and `MULTIPLY_ID`, respectively:

```
    let method_code = std::fs::read(MULTIPLY_PATH)
        .expect("Method code should be present at the specified path; did you use the correct *_PATH constant?");
    let mut prover = Prover::new(&method_code, MULTIPLY_ID)
        .expect("Prover should be constructed from valid method source code and corresponding method ID");

...

    receipt.verify(MULTIPLY_ID)
        .expect("Code you have proven should successfully verify; did you specify the correct method ID?");
```

# Intermission: Build and run the project!

In the main project folder, build and run the project using `cargo run --release`. Nothing exciting should happen yet, but we'll know if any of the above changes were missed. Use this command any time you'd like to check your progress.

# Step 3 (Host): Have the host program share two values with the guest

In this step, we'll be continuing to modify `factors/src/main.rs`. Let's start by picking some aesthetically pleasing primes:
```
fn main() {
    let a: u64 = 17;
    let b: u64 = 23;
```

Currently, our host driver program creates and runs a prover. The `prover.run()` command will cause our guest program to execute:

```
    let mut prover = Prover::new(&std::fs::read(MULTIPLY_PATH).unwrap(), MULTIPLY_ID).unwrap();

    let receipt = prover.run().unwrap();
```
 We'd like the host to make the values of `a` and `b` available to the guest prior to execution. Because the prover is responsible for managing guest-readable memory, we need to share them after the prover is created. To accomplish this, let's send our two values to the guest between the lines listed above:

 ```
    let mut prover = Prover::new(&std::fs::read(MULTIPLY_PATH).unwrap(), MULTIPLY_ID).unwrap();

    prover.add_input(to_vec(&a).unwrap().as_slice()).unwrap();
    prover.add_input(to_vec(&b).unwrap().as_slice()).unwrap();

    let receipt = prover.run().unwrap();
```


# Step 4 (Guest): Multiply two values and commit their result

Now it's time to start writing guest code. Open the main guest program file `methods/guest/src/bin/multiply.rs`. In its final form, we'll tell the guest to read the values of `a` and `b` from the host and multiply them together. We'll then publicly commit their product to the `receipt` portion of the `journal`.

Here is the complete guest program. We'll break this down step by step below:
```
pub fn main() {
    // Load the first number from the host
    let a: u64 = env::read();
    // Load the second number from the host
    let b: u64 = env::read();
    // Verify that neither of them are 1 (i.e. nontrivial factors)
    if a == 1 || b == 1 {
        panic!("Trivial factors")
    }
    // Compute the product while being careful with integer overflow
    let product = a.checked_mul(b).expect("Integer overflow");
    env::commit(&product);
}
```
## Load values from the host

First, we use `env::read()` to load both numbers:

```
    let a: u64 = env::read();
    let b: u64 = env::read();
}
```
## Confirm that factors are non-trivial

Next, we'll add a line that panics if either chosen value is 1. This will leave us with a guest program that only completes if the product of `a` and `b` is genuinely composite. 
```
    // Verify that neither of them are 1 (i.e. nontrivial factors)
    if a == 1 || b == 1 {
        panic!("Trivial factors")
    }
```

Now we can compute their product and `commit` it. Once committed to the `journal`, anyone with the receipt can read this value. As a final step, we'll have the host read and print the receipt's `journal` contents. In a real-world scenario, we'd want to hand the receipt to someone else, but reading it ourselves is a nice way to check our "Hello, Multiply!" project.

# Step 5 (Host): Generate a receipt and read its journal contents

For this step, we return to the main file for the host driver program at `factors/src/main.rs`, which currently ends with `receipt` generation after the prover runs:

```
    let receipt = prover.run().unwrap();
```

Now that we have a value to read from the receipt, let's extract the journal's contents. Below the line above, add the following lines.

```
    // Extract journal of receipt (i.e. output c, where c = a * b)
    let c: u64 = from_slice(&receipt.get_journal_vec().unwrap()).unwrap();

    // Print an assertion
    println!("Hello, world! I know the factors of {}, and I can prove it!", c);
```

If your program printed the "Hello, world!" assertion and `receipt` verification was a success, congratulations! If not, we hope that troubleshooting will get you familiar with the system, and we'd love to chat with you on [Discord](https://discord.com/invite/risczero). If you're ready to start building more complex projects, we recommend taking a look at the other examples in our [Getting Started resources](https://www.risczero.com/docs/) for more project ideas that use zero-knowledge proofs.