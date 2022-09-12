---
slug: password-checker
title: Checking Password Validity With the RISC Zero zkVM
---

We recommend pairing this document with the [password validity checker](https://github.com/risc0/password_checker) example in our Rust examples repository. For this example, we assume a high-level understanding of RISC Zero zkVM project design; if you'd like to read an example that describes a zkVM project in greater operational detail, we recommend the [Rust starter project](../examples/rust_starter.md) with accompanying [source code](https://github.com/risc0/risc0-rust-starter).

In this example, we'll discuss the security value of running a program as part of a RISC Zero zkVM project. 
(For a more detailed introduction, see the [zkVM overview](http://risczero.com/docs/explainers/zkvm/zkvm_overview).)
We'll take the perspective of Bob's Identity Service, which needs to set up authentication credentials for Alice. By the time you're finished reading this explanation, you should be able to broadly identify how Alice and Bob are relying on the RISC Zero zkVM.

### Fine Print (Or: Please Don't Use This In Production)

For those looking to implement their own password solutions: we do not present a complete implementation of a password validity checking program, nor is our example one that follows [guidelines recommended by NIST](https://pages.nist.gov/800-63-FAQ/#q-b06), which caution "against the use of composition rules (e.g., requiring lower-case, upper-case, digits, and/or special characters) for memorized secrets". In addition, in order for our implementation to be complete, Bob's Identity Service must further process Alice's resulting SHA256 salted password hash using a KDF such as scrypt before storing it in a database. This example also lacks the means to confirm that Alice is the actor responsible for generating her new password. `Our purpose here is to illustrate the power of sharing the results of private computations`; this program should not be incorporated into a password solution.

# Overview

In some ways, Alice's process follows convention. Alice generates a `password` that meets Bob's requirements, and Bob stores a `SHA-256 hash of Alice's password` along with a `salt`. However, by taking advantage of the RISC Zero zkVM, Alice can run the validity check and salted password hashing herself.

Like all RISC Zero projects for the zkVM, the bulk of the password checker program is divided between a [host driver](https://github.com/risc0/password_checker/blob/main/starter/src/main.rs) that runs the zkVM code and a [guest program](https://github.com/risc0/password_checker/blob/main/methods/guest/src/bin/pw_checker.rs) that executes on the zkVM. (If you're new to this process, we recommend the [Rust starter project](../examples/rust_starter.md).) Alice's process is as follows:

* Alice's `host driver program` shares a password and salt with the `guest zkVM` and initiates guest program execution. 
* The `guest zkVM program` checks Alice's password against a set of validity requirements (in our example, the password [must include digits and special characters](#some-important-fine-print)).
* If the password is valid, it is hashed with the provided salt using SHA-256. If the password does not meet validity requirements, the program panics and stops execution; no computational receipt is generated.
* Otherwise, the guest program generates a salted hash of Alice's password and commits it to a `journal`, part of a computational [`receipt`](../explainers/proof-system/what_is_a_receipt.md).
* Alice sends the receipt to Bob's Identity Service.

# What Information Can A Zero-Knowledge Proof Provide?

The proof receipt provides Bob with two important kinds of information. It demonstrates that the guest zkVM program has executed, which tells Bob `his password requirements were met`. It also provides a tamper-proof receipt `journal` for public outputs, the integrity of which tells Bob that `the shared outputs are the result of the password program`.

### Program Execution as Information

Rather than Alice allowing Bob's Identity Service to check her candidate password, Alice makes a zero-knowledge argument that she executed Bob's password-checking program. Bob can confirm his program was executed, and he knows the conditions that were required for his program to complete, so he knows that those conditions have been met.

### Linking Computations to Results

Instead of providing Bob's Identity Service with a plaintext password, Alice sends a zkVM `computational receipt` that discloses the hashed password and chosen hash salt. Without zero-knowledge technology, it is often complicated to trust that results from arbitrary third parties are related to computations they've performed. By verifying the RISC Zero zkVM receipt, Bob knows which guest program Alice has executed, and he knows that Alice's shared password hash is the result of having executed her guest program.

# RISC Zero zkVM Mechanics

In this section, we discuss <i>how</i> the RISC Zero zkVM allows Alice to produce a `computational receipt` that convinces Bob she has run the password setup program and shared its results.

### Constraint Checking and Seal Construction

When the host driver runs a guest program on the RISC Zero zkVM, its operations create an `execution trace`. Portions of the trace are subject to `constraints` that range from expectations about the RISC-V instruction set to assertions that memory has not been altered between instruction cycles. For more details, see [this explanation of seal construction](../explainers/proof-system/constructing-a-seal.md). Importantly, <b>these constraints are checked in a way that does not disclose the zkVM's operations or contents</b>.

* <b>What this means for Bob:</b> Without any insight into zkVM behavior, someone (possibly Alice) might modify how the guest zkVM performed its operations. Bob is confirming that the resulting operations follow certain rules (such as those governing how RISC-V instructions change memory register contents).
<br/>
<br/>
* <b>What this means for Alice:</b> if this argument of integrity were not zero-knowledge, then the proof of computation might reveal her password. We can think of the entire constraint-checking process as a zero-knowledge audit of the RISC Zero zkVM's computations, one in which Alice doesn't need to share her virtual machine's states or transitions.

### Comparing the Program Binary to Executed Instructions

The RISC Zero `zero-knowledge argument of computational integrity` asserts that Alice's guest zkVM carried out instructions as we might expect. Taken on its own, however, an assertion of integrity doesn't tell Bob <i>which</i> program instructions have been executed. If integrity checks stopped here, Alice might run a modified program that checks her password but substitutes an alternate, attacker-chosen password before generating a hash.

To confirm which program was executed, receipt validation also checks that an execution path is possible given a particular binary. To validate the receipt, Bob passes in his expected program's `method ID`. (Technically, a `method ID` is a table of roots for Merkle trees covering portions of the loaded ELF binary, but we can think of it like a lookup table for possible program instructions.) Checking the receipt against the `method ID` will either result in a match or not:
* if Alice executed Bob's program, then the loaded instructions her guest zkVM exercised will match some root in the method ID.
* If Alice's execution path is not possible given the expected program, then validation will fail.

### Shared Results: the Journal and the Seal

So far, Bob has enough information to confirm which program Alice executed and to know that execution followed the zkVM's constraint rules. In order to make use of Alice's shared salt and password, he must also trust the integrity of Alice's publicly shared values, stored in the receipt's `journal`. In other words, Bob needs to know that the journal contents have not been altered. We provide two assurances:
* First, a hash of the journal is included in the receipt's cryptographic `seal`, so altering the journal requires overwriting seal contents. 
* The seal itself is verified by the zk-STARK proof system and modifying the journal hash causes seal verification to fail.

The seal integrity requirements help protect against an attack that alters journal contents and then replaces the journal's hash.

# Deterministic Programs: A Design Note

We now know that Bob can confirm a program's identity, computational integrity, and shared results by checking Alice's computational `receipt`. As a final precaution, we'd like to ensure that these checks don't accidentally reveal Alice's password. To understand this consideration, let's return briefly to the receipt verification process.

Bob identifies Alice's program using a `method ID` whose uniqueness is tied to the uniqueness of the guest program. Anyone with a valid `receipt` from Alice can therefore make guesses about the program she executed (much like guessing a password by computing candidate password hashes). Therefore, neither her program's identity nor its contents should be a secret. <b>Guest code should never include any deterministic values that we do not expect someone to share</b>.

Furthermore, <b>guest zkVM code cannot include any content that we'd expect someone to regularly change</b>, which includes configuration files and hard-coded values. In the case of Bob and Alice, Alice's guest zkVM password-checking program does not vary between password choices because the password and salt are loaded into guest memory by the host. Supplying the password and salt outside the guest zkVM binary not only helps protect Alice's secrets, it also ensures that Bob's Identity Service expects one unique `method ID` for its users.

# Writing for the RISC Zero zkVM

This is not an exhaustive security model and the RISC Zero zkVM is under early active development. However, we strongly believe that this technology enables fundamentally different kinds of software solutions to hard problems. We hope that this example has given you more insights into how the zkVM proof system works.

If this has given you any project ideas or you'd like to talk with us about secure design, come join the conversation on the [RISC Zero Discord](https://discord.gg/risczero).