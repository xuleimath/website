---
slug: metamath
title: Running a Metamath Checker inside a zkVM
authors: [bolton]
tags: [metamath, fm, zk]
---

Hello everyone! I'm Bolton Bailey, a Ph.D. student and software
engineer at RISC Zero. I'm writing this blog post to discuss a project
that I've been working on, in conjunction with Runtime Verification
Inc., to run a Metamath checker inside the RISC Zero zkVM. Although this
project is just getting started, I think it's headed in a very exciting
direction. In this post, I'd like to share with you all what makes it
so interesting, how we've gone about it, and what it could look like
going forward.

## What are Formal Methods?

My field of research lies at the intersection of cryptography and formal
methods. Formal methods are a mathematically rigorous approach to
software development that ensures a piece of code does what it is
supposed to do. A formal methods approach goes farther than simply
examining a piece of code line-by-line or writing a test suite to ensure
the code runs correctly on a number of examples. Instead, we write out a
mathematically precise specification of what the code is supposed to do
and prove that the code will always behave as specified, no matter what
input it is given.

Rather than writing down specifications and proofs on paper and checking
them by hand, we get the computer itself to help us. Formalists use
specialized computer languages known as formal systems to express their
proofs, and they write programs to check these proofs so that they can
be sure they made no mistakes. The language I've been working with is
the Metamath system.

### What is Metamath?

[Metamath](https://us.metamath.org/) is a formal system that's
designed to be flexible enough to represent proofs in a variety of
mathematical frameworks, but also very simple for the computer to
understand. The way a computer processes logic can be unintuitive for a
human, but this section will hopefully give you an idea of how it works.

Here is an example of some simple Metamath code from [the
manual](https://us.metamath.org/downloads/metamath.pdf). This code
describes a simple mathematical system for talking about numbers, with
addition and zero as the basic concepts:

```mm
$( Declare the constant symbols we will use $)
$c 0 + = -> ( ) term wff |- $.
$( Declare the metavariables we will use $)
$v t r s P Q $.
$( Specify properties of the metavariables $)
tt $f term t $.
tr $f term r $.
ts $f term s $.
wp $f wff P $.
wq $f wff Q $.
$( Define "term" and "wff" $)
tze $a term 0 $.
tpl $a term ( t + r ) $.
weq $a wff t = r $.
wim $a wff ( P -> Q ) $.
$( State the axioms $)
a1 $a |- ( t = r -> ( t = s -> r = s ) ) $.
a2 $a |- ( t + 0 ) = t $.
$( Define the modus ponens inference rule $)
${
  min $e |- P $.
  maj $e |- ( P -> Q ) $.
  mp $a |- Q $.
$}
$( Prove a theorem $)
th1 $p |- t = t $=
$( Here is its proof: $)
tt tze tpl tt weq tt tt weq tt a2 tt tze tpl
tt weq tt tze tpl tt weq tt tt weq wim tt a2
tt tze tpl tt tt a1 mp mp
$.
```

There is a lot of boilerplate, but the critical part of the definition
is the two axioms listed on the lines that start with `a1` and `a2`. Respectively, these axioms state that:

1. If `t = r` and `t = s`, then `r = s`.
2. For any term `t`, `t + 0 = t`.

As an example of how detailed these proofs can get, let's unpack the
very last theorem at the end of the example above, which proves `t = t`
for any term `t`. The proof consists of a very low-level list of
invocations of rules and axioms that show the theorem holds. At a high
level, the proof says that if we substitute `t` from the first axiom above
with `t + 0`, and `r` and `s` are substituted for `t`, then the first axiom
becomes "If `t + 0 = t` and `t + 0 = t`, then `t = t`". By the second axiom
listed above, the `t + 0 = t` conditions hold; therefore, the first axiom
is telling us that `t = t`. Notice that we have to use the first axiom
once and the second axiom twice (once for each condition of the first
axiom). While not all of the symbols in the proof of `th1` might be
familiar, we can see it includes one reference to `a1` and two references
to `a2`.

This may seem like an overly convoluted way of proving a very simple
fact. This is because computers need to have every detail spelled out to
them. Metamath in particular is designed to not have a lot of built-in
optimizations, in order to keep the Metamath checking program as simple
as possible. Unlike a human, the Metamath needs to be reminded of the
fact that "`t` is a number" every time `t` comes up again in the argument.
This is essentially what the `tt` token does - you can see it comes up 15
times in the proof of `th1`!

### What are the possibilities for formal methods?

The example theorem above is a simple one. In principle, however,
there's no limit on what can be proven within a formal system like
Metamath: we can describe logical systems that deal with set theory or
even outline the semantics of programming languages. Within formal
methods, programs and data are just different types of mathematical
objects, ones for which confidence can be of the utmost importance. The
software industry in general, and the cryptocurrency industry in
particular, has a lot at stake when it comes to bugs in their systems:
[*billions of
dollars*](https://www.nytimes.com/2022/09/28/technology/crypto-hacks-defi.html)
have been hacked out of blockchains, and
[*entire*](https://runtimeverification.com/)
[*organizations*](https://galois.com/) exist that look to use formal
analysis to help plug these holes.

## Zero-Knowledge Proofs for Formal Methods

Having established what formal methods are and why they're useful,
let's ask why someone might want to use succinct zero-knowledge proofs
in conjunction with them.

Suppose a technology firm wants to be assured that their products have
no bugs. They open a bug bounty and offer a prize to anyone who can find
a flaw. A formal analysis firm writes a specification of what the code
is supposed to do and then attempts to find a proof that the code meets
the specification. In their analysis, they end up finding a bug, and
they create a formal proof that the specification is not satisfied.

At this point, the formalist firm is nervous that the software writers
will look at the proof and say that actually, the bug isn't a problem
at all, and this doesn't merit paying out the bug bounty. Besides, the
technology firm doesn't want to run this long formal check themselves.
It might not even be a technology firm offering the bounty at all, but
instead a smart contract.The smart contract is limited in how much gas
it can use, so it can't reasonably check the whole formal analysis.

This scenario is ripe for a succinct proof application. The formalists
can prove the bug is real without a big data transfer or computation by
sending a STARK proof that the formal proof check validated their code
in Metamath or another formal system. As a bonus, they can make the
STARK a zk-STARK if they are worried about someone seeing the proof,
reverse-engineering the bug, and making an exploit.

## Making it a Reality with RISC Zero

How does all this work? In a succinct argument system like a zk-STARK,
we have a prover and a verifier. The prover is meant to prove that the
outcome of some computation gave some result, and the verifier checks
the proof is correct. These two roles are analogous to the formal proof
writer and the formal proof checker. In the case of this application,
they coincide completely!

The prover first makes a Metamath file to prove their theorem. They then
run a Metamath checker for the file as a computation inside the ZK proof
system. This gives them a cryptographic output proof that they then pass
to the verifier. In order to make this work, we need to be able to
encode the Metamath proof checking process inside a zkVM.

This is where RISC Zero enters the picture. If we were doing this
project in a zkEVM or using a circuit model, it would be necessary to
write a Metamath checker in Solidity, Circom, or some other specialized
language. But because RISC Zero has a zkVM for the RISC-V architecture,
I can write my Metamath checker in a language like Rust. In fact, **I
don't even need to know Rust, because a Metamath checker written in
Rust already exists, and I can just drop it into the RISC Zero VM and
have it work!** Even if formal methods aren't of interest to you at
all, the implications for making STARK cryptography accessible are
profound and exciting.

## The Repository

You can find the GitHub repository for our project
[here](https://github.com/BoltonBailey/risc0-metamath). I exaggerate a
little when I say you don't even have to know Rust to make a Rust
program run in the RISC zero zkVM, but not by much. The repository is a
fork of the much simpler
[risc0/rust-starter](https://github.com/risc0/risc0-rust-starter),
which has the same basic structure. There are two Rust programs in each:
a host that sends data to the zkVM through the add_input() method and a
guest that runs inside the zkVM, receives input via the env::read()
method, and commits to outputs using the env::commit() method.

When you're designing a zk-STARK, it's good to think about what data
is part of the *statement* and what is part of the *witness*. The
statement is the data that is known to the verifier. In the case of our
Metamath verifier, it includes the statement of the theorem that is
being proven. It also includes the list of the axioms on which that
theorem is based (otherwise, the prover might create their own axiom
which says that the theorem is true). The witness is all the data that
is known to the prover. In our case, this is simply an entire Metamath
file, along with an indication of which of the theorems within that file
is the one for which we are requesting the proof.

Our project has been able to run a variety of small Metamath programs,
such as the one in the example above. Our primary bottleneck is the
execution time within the VM; because producing a STARK proof requires
memory proportional to the size of the execution trace, we can only
check Metamath files of a few hundred lines before we hit the limit. The
RISC Zero team has been working hard though, and I suspect we will soon
have engineering solutions to remove this obstruction. From there, the
only limit on the space of STARKed formal methods will be our own
imaginations!
