---
slug: announce
title: A New Day for ZK
authors: [brianretford]
tags: [zk, oss]
---

We are very excited to share a new kind of computing platform with the world – 
*one that can't lie about anything it does.* RISC Zero enables *any developer*
to build zero-knowledge proofs that can be *executed and verified on any modern
computer* in the languages they are most familiar with.

RISC Zero is the first fully **free-and-open-source general-purpose ZK computing
platform**. Our **prover and verifier** are [available under the Apache2
license](https://github.com/risc0/risc0), along with several simple but powerful
examples of this technology 
[written in Rust](https://github.com/risc0/risc0/tree/main/examples/rust) and 
C++.

![Logo Blue](./logoblue.png)

# General Purpose Verifiable Computing

RISC Zero is a fully-compliant software implementation of the RISC-V 
microarchitecture (TODO exact designator) implemented in a zk-STARK based
proving system.

### So is it what exactly?

RISC Zero creates a zero-knowledge virtual machine that can run on any platform.
It's like a small microcontroller or co-processor that produces receipts for 
every program it runs. These receipts can be verified at anytime by anyone 
in ~constant time – **even if you don't trust the party that executed the code**
.