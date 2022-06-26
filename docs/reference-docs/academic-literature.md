# Academic Literature
*This page contains a list of academic references that RISC Zero builds on.*

At the core of RISC Zero's technology is a **zk-STARK** (`zero-knowledge, Scalable, Transparent, ARgument of Knowledge`) implemented over a Von Neumann architecture. 

The world of zero-knowledge computing is quite new to real-world applications, but the underlying academic discourse about zero-knowedge proofs (ZKPs) has been vibrant for many years. 

If you're new to these ideas (and comfortable with very technical writing), the [STARK paper](https://eprint.iacr.org/2018/046.pdf) and our [STARKS reference page](about-starks.md) are a good place to start. 

## Key Papers 
These are the papers that make RISC Zero possible. We are grateful for the incredible work put forth by these thinkers.
- [Proximity of Reed Solomon Gaps (Ben Sasson et al, 2020)](https://eprint.iacr.org/2020/654.pdf)
- [The PLONK Paper (Gabizon et al, 2019)](https://eprint.iacr.org/2019/953.pdf)
- [The DEEP-FRI paper (Ben-Sasson et al, 2019)](https://arxiv.org/pdf/1903.12243.pdf)
- [The STARK paper (Ben-Sasson et al, 2018](https://eprint.iacr.org/2018/046.pdf)
- [The FRI paper (Ben-Sasson et al, 2018)](https://drops.dagstuhl.de/opus/volltexte/2018/9018/pdf/LIPIcs-ICALP-2018-14.pdf)

## Historical Underpinnings
Of course, each of the papers above has its own origin story. The following papers provide some of the historical progression of ideas that paved the path to where we stand now. 
- [Zero Knowledge Proofs of Proximity (Berman, Rothblum, Vaikuntanathan, 2017)](https://eprint.iacr.org/2017/114.pdf)
- [Interactive Oracle Proofs (Ben Sasson et al, 2015)](https://www.iacr.org/archive/tcc2016b/99850156/99850156.pdf)
- [Interactive Proofs of Proximity (Rothblum et al, 2014)](https://guyrothblum.files.wordpress.com/2014/11/rvw13.pdf)
- [The PCP Theorem by Gap Amplification (Dinur, 2007)](https://www.wisdom.weizmann.ac.il/~dinuri/mypapers/combpcp.pdf)
- [The PCP Theorem (Arora and Safra, 1998)](https://www.cs.umd.edu/~gasarch/TOPICS/pcp/AS.pdf)
- [How to Prove Yourself (Fiat-Shamir, 1986)](https://link.springer.com/content/pdf/10.1007/3-540-47721-7_12.pdf)

- [The Original Zero-Knowledge Paper (Goldwasser, Micali, and Rackoff, 1985)](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.419.8132&rep=rep1&type=pdf)
- [The Merkle Tree paper (Merkle, 1979)](https://people.eecs.berkeley.edu/~raluca/cs261-f15/readings/merkle.pdf)
- [The Reed-Solomon paper (Reed and Solomon, 1960)](https://faculty.math.illinois.edu/~duursma/CT/RS-1960.pdf) 
- [The Hamming code paper (Hamming, 1950)](http://vtda.org/pubs/BSTJ/vol29-1950/articles/bstj29-2-147.pdf) 

## Subtleties of the Fiat-Shamir Heuristic
Because a poor implementation of Fiat-Shamir is responsible for so many security problems in zero-knowledge applications, we include these papers which inform our understanding of how to safely use this technique. 
- [Fiat Shamir Transformation of Multi-Round Interactive Proofs (Attema et al, 2021. Updated 2022)](https://eprint.iacr.org/2021/1377.pdf)

- [Re-visiting Post-Quantum Fiat-Shamir (Liu and Zhandry, 2019)](https://eprint.iacr.org/2019/262.pdf)
- [Security of the Fiat-Shamir Transformation in the Quantum Random-Oracle Model (Don et al, 2019)](https://arxiv.org/pdf/1902.07556.pdf)

- [How Not to Prove Yourself (Bernhard et al, 2014)](http://www.uclouvain.be/crypto/services/download/publications.pdf.87e67d05ee05000b.6d61696e2e706466.pdf)
- [On the Insecurity of Fiat-Shamir (Goldwasser and Tauman, 2004)](https://eprint.iacr.org/2003/034.pdf)
