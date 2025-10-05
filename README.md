# Project Proposal
Authors: 
- Parisa Betel Miri (parisa.betel.miri@mail.utoronto.ca)
- Angela Yu (angelaxh.yu@mail.utoronto.ca)

## Abstract

For this project we’ll be porting parallel C++ benchmarks from the PBBS (link) to Rust. The main motivations for this are two-fold, to better understand the strengths and weaknesses of Rust’s unique approach to parallel programming, and to expand the ecosystem of available benchmarks.

## 1. Background and Motivation

A key selling point of the Rust programming language is that of “fearless concurrency”, where in safe Rust allowing only immutable references, or a single mutable reference can prevent a number of common pitfalls of parallel programming, such as data races and deadlocks. However, for some applications these rules are overly restrictive and thus we need unsafe Rust. Some such cases can take advantage of existing unsafe libraries. For example Rayon [P] is written in unsafe Rust, but this library has been used hundreds of millions of times so it’s very well tested. Therefore, an application developer might not have to write their own unsafe code. Clearly, developers will experience a variable degree of safety depending on the problem at hand.

A set of benchmarks can be a valuable asset with which compiler researchers can test the correctness and performance impacts of compiler modifications. As a mature programming language, C++ has a wealth of such benchmarks, whereas Rust is still lacking. On the C++ side, there are a variety of benchmark suites with different parallel patterns and are designed for various C++ parallel libraries. For example, below is a not comprehensive list of C++ benchmarks:
 
1. The Problem-Based Benchmark Suite (PBBS) [D, Q]: a collection of 22 parallel algorithms, such as sort and graph algorithms. 
2. NAS Parallel Benchmarks (NPB) [C]: 19 benchmarks derived from real-world applications with various computation and data movement patterns to evaluate high-performance parallel supercomputers.
3. Princeton Application Repository for Shared-Memory Computers (PARSEC) [E, F]: 12 benchmarks for evaluating multiprocessors based on real-world applications with different parallelization patterns and data usage patterns.
4. pSTL-Bench [G, H] and parSTL [A, B]: for evaluating parallel algorithms in C++ Standard Template Library (STL). 

On the other hand, research on Rust parallel benchmarks is still in its early stages. Prior research has been porting C++ benchmarks to Rust to help fill the gap of lack of parallel benchmarks in the Rust ecosystem. For example, Rust Parallel Benchmarks suite (RPB) [I, J] ported 12 benchmarks from C++ PBBS suite, and Rust-NPB [K, L] ported 8 benchmarks from C++ NPB suite. Both RPB and Rust-NPB used the Rayon crate to implement the Rust version of their benchmark suite. Other researchers created their own Rust benchmarks, for example, the Rust Stream Bench [M, N] provides 4 steaming benchmarks. 

Seeing the gap in the Rust ecosystem, we are motivated to extend and contribute to the Rust parallel benchmark suites.


## 2. Objectives 

In this project, our first goal is to contribute to the Rust ecosystem by continuing on existing Rust benchmark research to port more C++ parallel benchmarks to Rust. In particular, we will be focusing on extending the RPB suite by porting more parallel benchmarks from C++ PBBS suite to cover a broader range of parallelism patterns, and looking to improve existing benchmarks that are underperforming. There are other similar benchmark suites, but PBBS serves as a good domain-neutral set, some of which has already been ported.

Our next goal is to better understand the strengths and limitations of parallel programming in Rust, and how a programmer might navigate the aforementioned tradeoffs. In particular, we want to see what can be implemented in safe Rust, what can be done with unsafe libraries, and what requires custom unsafe code.

Another objective is to compare the Rust benchmarks’ performance with their C++ version. We are curious if the “fearless concurrency” language Rust and the powerful Rayon crate can naturally bring performance improvements through programming in a “Rusty” way, or if the relative maturity of the C++ toolchains give it the edge. Although the Rust compiler’s error messages are quite sophisticated, the optimization passes of modern compilers are enormously complex, and we want to see how Rust fairs in this area. Authors of Rust-NPB compared the performance of Rust and C++ finding Rust was 5.6% faster than C++ sequentially, but 19.0% slower when run in parallel [K]. Similarly, the existing RPB benchmarks were 9% faster sequentially, but 44% slower in parallel [O]. We’ll see how our results compare to these.

## 3. Tentative Plan

In this project, our first goal is to contribute to the Rust ecosystem by continuing on existing Rust benchmark research to port more C++ parallel benchmarks to Rust. In particular, we will be focusing on extending the RPB suite by porting more parallel benchmarks from C++ PBBS suite to cover a broader range of parallelism patterns, and looking to improve existing benchmarks that are underperforming. There are other similar benchmark suites, but PBBS serves as a good domain-neutral set, some of which has already been ported.

Our next goal is to better understand the strengths and limitations of parallel programming in Rust, and how a programmer might navigate the aforementioned tradeoffs. In particular, we want to see what can be implemented in safe Rust, what can be done with unsafe libraries, and what requires custom unsafe code.

Another objective is to compare the Rust benchmarks’ performance with their C++ version. We are curious if the “fearless concurrency” language Rust and the powerful Rayon crate can naturally bring performance improvements through programming in a “Rusty” way, or if the relative maturity of the C++ toolchains give it the edge. Although the Rust compiler’s error messages are quite sophisticated, the optimization passes of modern compilers are enormously complex, and we want to see how Rust fairs in this area. Authors of Rust-NPB compared the performance of Rust and C++ finding Rust was 5.6% faster than C++ sequentially, but 19.0% slower when run in parallel [K]. Similarly, the existing RPB benchmarks were 9% faster sequentially, but 44% slower in parallel [O]. We’ll see how our results compare to these.

## References