# Project Proposal
Authors: 
- Parisa Betel Miri (parisa.betel.miri@mail.utoronto.ca)
- Angela Yu (angelaxh.yu@mail.utoronto.ca)

## Abstract

For this project we’ll be porting parallel C++ benchmarks from the PBBS (link) to Rust. The main motivations for this are two-fold, to better understand the strengths and weaknesses of Rust’s unique approach to parallel programming, and to expand the ecosystem of available benchmarks.

## 1. Background and Motivation



## 2. Objectives 

In this project, our first goal is to contribute to the Rust ecosystem by continuing on existing Rust benchmark research to port more C++ parallel benchmarks to Rust. In particular, we will be focusing on extending the RPB suite by porting more parallel benchmarks from C++ PBBS suite to cover a broader range of parallelism patterns, and looking to improve existing benchmarks that are underperforming. There are other similar benchmark suites, but PBBS serves as a good domain-neutral set, some of which has already been ported.

Our next goal is to better understand the strengths and limitations of parallel programming in Rust, and how a programmer might navigate the aforementioned tradeoffs. In particular, we want to see what can be implemented in safe Rust, what can be done with unsafe libraries, and what requires custom unsafe code.

Another objective is to compare the Rust benchmarks’ performance with their C++ version. We are curious if the “fearless concurrency” language Rust and the powerful Rayon crate can naturally bring performance improvements through programming in a “Rusty” way, or if the relative maturity of the C++ toolchains give it the edge. Although the Rust compiler’s error messages are quite sophisticated, the optimization passes of modern compilers are enormously complex, and we want to see how Rust fairs in this area. Authors of Rust-NPB compared the performance of Rust and C++ finding Rust was 5.6% faster than C++ sequentially, but 19.0% slower when run in parallel [K]. Similarly, the existing RPB benchmarks were 9% faster sequentially, but 44% slower in parallel [O]. We’ll see how our results compare to these.

## 3. Tentative Plan

In this project, our first goal is to contribute to the Rust ecosystem by continuing on existing Rust benchmark research to port more C++ parallel benchmarks to Rust. In particular, we will be focusing on extending the RPB suite by porting more parallel benchmarks from C++ PBBS suite to cover a broader range of parallelism patterns, and looking to improve existing benchmarks that are underperforming. There are other similar benchmark suites, but PBBS serves as a good domain-neutral set, some of which has already been ported.

Our next goal is to better understand the strengths and limitations of parallel programming in Rust, and how a programmer might navigate the aforementioned tradeoffs. In particular, we want to see what can be implemented in safe Rust, what can be done with unsafe libraries, and what requires custom unsafe code.

Another objective is to compare the Rust benchmarks’ performance with their C++ version. We are curious if the “fearless concurrency” language Rust and the powerful Rayon crate can naturally bring performance improvements through programming in a “Rusty” way, or if the relative maturity of the C++ toolchains give it the edge. Although the Rust compiler’s error messages are quite sophisticated, the optimization passes of modern compilers are enormously complex, and we want to see how Rust fairs in this area. Authors of Rust-NPB compared the performance of Rust and C++ finding Rust was 5.6% faster than C++ sequentially, but 19.0% slower when run in parallel [K]. Similarly, the existing RPB benchmarks were 9% faster sequentially, but 44% slower in parallel [O]. We’ll see how our results compare to these.

## References