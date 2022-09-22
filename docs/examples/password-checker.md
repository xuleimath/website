---
slug: password-checker
title: Checking Password Validity With the RISC Zero zkVM
---


In this example, we'll discuss the security value of running a program as part of a RISC Zero zkVM project. We recommend pairing this document with the [password validity checker](https://github.com/risc0/password_checker) example in our Rust examples repository. We assume a high-level understanding of RISC Zero zkVM project design; if you'd like to read an example that describes a zkVM project in greater operational detail, we recommend the [Rust starter project](../examples/rust_starter.md).

We'll take the perspective of Bob's Identity Service, which needs to set up authentication credentials for Alice. By the time you're finished reading this explanation, you should be able to broadly identify how Alice and Bob are relying on the RISC Zero zkVM.

### Fine Print (Or: Please Don't Use This In Production)

For those looking to implement their own password solutions: we do not present a complete implementation of a password validity checking program, nor is our example one that follows [guidelines recommended by NIST](https://pages.nist.gov/800-63-FAQ/#q-b06), which caution "against the use of composition rules (e.g., requiring lower-case, upper-case, digits, and/or special characters) for memorized secrets". Bob's Identity Service should responsibly further process Alice's resulting SHA256 salted password hash using a KDF such as scrypt before storing it in a database. This example has also not supplied the mechanism by which Alice confirms she is the actor responsible for generating her new password. <b>Our purpose here is simply to illustrate the power of sharing the results of private computations.</b>

# Overview

In some ways, Alice's process follows convention. Alice generates a `password` that meets Bob's requirements, and Bob receives a `SHA-256 hash of Alice's password` along with a `salt`. Like all RISC Zero projects for the zkVM, the bulk of the password checker program is divided between a [host driver](https://github.com/risc0/password_checker/blob/main/starter/src/main.rs) that runs the zkVM code and a [guest program](https://github.com/risc0/password_checker/blob/main/methods/guest/src/bin/pw_checker.rs) that executes on the zkVM. By taking advantage of the RISC Zero zkVM, Alice can run a password validity check and her password never needs to leave her local machine.Alice's process is as follows:

* Alice's `host driver program` shares a password and salt with the `guest zkVM` and initiates guest program execution.
* The `guest zkVM program` checks Alice's password against a set of validity requirements.
* If the password is valid, it is hashed with the provided salt using SHA-256. If not, the program panics and no computational receipt is generated.
* The guest program generates a salted hash of Alice's password and commits it to a `journal`, part of a computational [`receipt`](../explainers/proof-system/what_is_a_receipt.md).
* Alice sends the receipt to Bob's Identity Service.

# What Information Can A Zero-Knowledge Proof Provide?

The `method ID` and the the `journal` on the receipt provide Bob assurance that:

    The program Alice executed within the zkVM was actually Bob's Password Checker, and
    Bob's Password Checker approved Alice's password

* It demonstrates that the guest zkVM program has executed, which tells Bob `his password requirements were met`.
* It also provides a tamper-proof `journal` for public outputs, the integrity of which tells Bob that <b>the shared outputs are the result of running the password program</b>.

### Successful Program Execution is Information

Rather than allowing Bob's Identity Service to check her candidate password, Alice makes a zero-knowledge argument that she executed Bob's password-checking program. Bob can confirm his program was executed, and he knows the conditions that were required for his program to complete, so he knows that those conditions have been met.

### Linking Computations to Results

Instead of providing Bob's Identity Service with a plaintext password, Alice sends a `receipt` that discloses the hashed password and chosen hash salt. Without zero-knowledge technology, it is often complicated to trust that results from arbitrary third parties are related to computations they've performed. By verifying the  `receipt`, Bob knows which guest program Alice has executed, and he knows that Alice's shared password hash is the result of having executed her `guest program`.

# RISC Zero zkVM Mechanics

In this section, we discuss <i>how</i> the RISC Zero zkVM allows Alice to produce a `receipt` that convinces Bob she has run the password setup program and shared its results.

### Constraint Checking and Seal Construction

When the host driver runs a guest program on the RISC Zero zkVM, its operations create an `execution trace`. The trace is subject to `constraints` that include expected RISC-V instruction set behaviors and an expectation that memory register contents have not been modified out of turn. For more details, see [this explanation of seal construction](../explainers/proof-system/constructing-a-seal.md). Importantly, <b>these constraints are checked in a way that does not disclose the zkVM's operations or contents</b>.

<b>What this means for Bob</b>: The journal on Alice's receipt was generated by executing the code in Bob's password checker in accordance with the rules of RISC-V.
<br/>
<b>What this means for Alice</b>: She's able to satisfy Bob's password requirements while maintaining full privacy of her password: Alice's password never left her machine, and the receipt communicates no knowledge about her password (aside from the fact that it satisfies Bob's requirements).

### Comparing the Program Binary to Executed Instructions

Taken on their own, these constraints on computational integrity don't tell Bob <i>which</i> program instructions have been executed. If we stopped here, Alice might run a modified program that checked her password but substituted an alternate, attacker-chosen password prior to hash generation.

Bob's receipt validation process also confirms that the correct program was executed. Bob validates the receipt using a `method ID` associated with the program he expects Alice to run. If Bob's `receipt` is validated, then Alice's program execution matches Bob's password-checking program.

### Shared Results: the Journal and the Seal

So far, Bob has enough information to confirm which program Alice executed and to know that execution followed the zkVM's constraint rules. In order to make use of Alice's shared salt and password, he must also trust the integrity of Alice's publicly shared values, stored in the receipt's `journal`. In other words, Bob needs to know that the journal contents have not been altered. We provide two assurances:
* First, a hash of the journal is included in the receipt's cryptographic `seal`, so altering the journal requires overwriting seal contents. 
* The seal itself is verified by the zk-STARK proof system and modifying the journal hash causes seal verification to fail.

The seal integrity requirements help protect against an attack that alters journal contents and then replaces the journal's hash.

# Deterministic Programs: A Design Note

We now know that Bob can confirm a program's identity, computational integrity, and shared results by checking Alice's computational `receipt`. As a final precaution, we'd like to ensure that these checks don't accidentally reveal Alice's password. To understand this consideration, let's return briefly to the receipt verification process.

Bob identifies Alice's program using a `method ID` whose uniqueness is tied to the uniqueness of the guest program. Anyone with a valid `receipt` from Alice can therefore make guesses about the program she executed (much like guessing a password by computing candidate password hashes). Therefore, neither her program's identity nor its contents should be a secret. <b>Guest code should never include any deterministic values that we do not expect someone to share</b>. 

Relatedly, guest zkVM code also should not include any content that we'd expect someone to regularly change, which includes configuration files and hard-coded values. Alice's guest zkVM password-checking program does not vary between password choices because the password and salt are not included in the guest program's ELF binary; they are loaded into guest memory by the host. This not only helps protect Alice's secrets, it also ensures that Bob's Identity Service expects one unique `method ID` for all its users.

# Writing for the RISC Zero zkVM

This is not an exhaustive security model and the RISC Zero zkVM is under early active development. However, we strongly believe that this technology enables fundamentally different kinds of software solutions to hard problems. We hope that this example has given you more insights into how the zkVM proof system works.

If this has given you any project ideas or you'd like to talk with us about secure design, come join the conversation on the [RISC Zero Discord](https://discord.gg/risczero).