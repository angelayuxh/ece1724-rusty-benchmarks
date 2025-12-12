# Project Final Report
Authors: 
- Parisa Betel Miri (1012458653) - parisa.betel.miri@mail.utoronto.ca
- Angela Yu (1007004830) - angelaxh.yu@mail.utoronto.ca

## 1. Motivation

A key selling point of the Rust programming language is that of “fearless concurrency”, where in safe Rust allowing only immutable references, or a single mutable reference can prevent a number of common pitfalls of parallel programming, such as data races and deadlocks. However, for some applications these rules are overly restrictive and thus we need unsafe Rust. Some such cases can take advantage of existing unsafe libraries. For example Rayon [3] is written in unsafe Rust, but this library has been used hundreds of millions of times so it’s very well tested. Therefore, an application developer might not have to write their own unsafe code. Clearly, developers will experience a variable degree of safety depending on the problem at hand.

A set of benchmarks can be a valuable asset with which compiler researchers can test the correctness and performance impacts of compiler modifications. As a mature programming language, C++ has a wealth of such benchmarks, whereas Rust is still lacking. On the C++ side, there are a variety of benchmark suites with different parallel patterns and are designed for various C++ parallel libraries. For example, below is a non-comprehensive list of C++ benchmarks:
 
1. The Problem-Based Benchmark Suite (PBBS) [1, 2]: a collection of 22 parallel algorithms, such as sort and graph algorithms. 
2. NAS Parallel Benchmarks (NPB) [4]: 19 benchmarks derived from real-world applications with various computation and data movement patterns to evaluate high-performance parallel supercomputers.
3. Princeton Application Repository for Shared-Memory Computers (PARSEC) [5, 6]: 12 benchmarks for evaluating multiprocessors based on real-world applications with different parallelization patterns and data usage patterns.
4. pSTL-Bench [7, 8] and parSTL [9, 10]: for evaluating parallel algorithms in C++ Standard Template Library (STL). 

On the other hand, research on Rust parallel benchmarks is still in its early stages. Prior research has been porting C++ benchmarks to Rust to help fill the gap of lack of parallel benchmarks in the Rust ecosystem. For example, Rust Parallel Benchmarks suite (RPB) [11, 12] ported 12 benchmarks from C++ PBBS suite, and Rust-NPB [13, 14] ported 8 benchmarks from C++ NPB suite. Both RPB and Rust-NPB used the Rayon crate to implement the Rust version of their benchmark suite. Other researchers created their own Rust benchmarks, for example, the Rust Stream Bench [15, 16] provides 4 steaming benchmarks. 

Seeing the gap in the Rust ecosystem, we were motivated to extend and contribute to the Rust parallel benchmark suites.

## 2. Objectives 

In this project, our first goal was to contribute to the Rust ecosystem by continuing on existing Rust benchmark research to port more C++ parallel benchmarks to Rust. We focused on extending the RPB suite by porting more parallel benchmarks from C++ PBBS suite to cover a broader range of parallelism patterns There are other similar benchmark suites, but PBBS serves as a good domain-neutral set, some of which has already been ported. The longer term objective, beyond this course project, is that RPB achieves parity with PBBS.

Our next goal was to better understand the strengths and limitations of parallel programming in Rust, and how a programmer might navigate the aforementioned tradeoffs. In particular, we wanted to see what can be implemented in safe Rust, what can be done with unsafe libraries, and what requires custom unsafe code.

Another objective was to compare the Rust benchmarks’ performance with their C++ version. We were curious if the “fearless concurrency” language Rust and the powerful Rayon crate could naturally bring performance improvements through programming in a “Rusty” way, or if the relative maturity of the C++ toolchains give it the edge. Although the Rust compiler’s error messages are quite sophisticated, the optimization passes of modern compilers are enormously complex, and we wanted to see how Rust fairs in this area. Authors of Rust-NPB compared the performance of Rust and C++ finding Rust was 5.6% faster than C++ sequentially, but 19.0% slower when run in parallel [13]. Similarly, the existing RPB benchmarks were 9% faster sequentially, but 44% slower in parallel [17]. We’ll see how our results compare to these.

## 3. Features

The PBBS benchmark suite is built on ParlayLib [18] which is a parallel algorithm library developed in C++. The original authors already ported part of the ParlayLib functions into Rust; we further contributed to the functions that are missing in the original RPB. We ported a total of 4 benchmarks from text processing and computational geometry/graphics categories, along with their required ParlayLib functions. 

### 3.1 Text Processing

Two benchmarks under the text processing category are ported, namely Word Count and Inverted Index. We ported a sequential version and a parallel version for each of them.

#### 3.1.1 Word Count

Word count returns the number of occurrences of each word in the input text file. All non-alphabetical characters will be replaced as spaces and treated as word delimiters. 

**Sequential**

In the sequential version, the input text is chopped into tokens using the native Rust function `split_ascii_whitespace`. Then, we used Rust’s hash map to count the occurrence of each token, where the key is the word/token, and the value is the occurrence counter. Instead of using the native hash function, we replaced it with the djb2 hash used by the original PBBS benchmark to ensure a fair performance comparison. 

**Parallel**

The parallel version of word count replaced the sequential tokenization and writing into hash map with parallelized algorithms from ParlayLib, `tokens` and `histogram_by_key`, respectively. We also used parallel iterators from the Rayon library to enable parallelized access on different data structures whenever possible. 

We ported the function `tokens` and its required sub-functions into Rust. `tokens` calls the function `map_tokens` internally. `map_tokens` performs parallel splitting of the input sequence of characters into words, with the default delimiter being whitespaces, and applies the function `f` on each word/token. In this case, `f` is set to directly return the underlying token. `map_tokens` first marks every token start, then performs a parallel prefix-scan (by calling `block_delayed_scan`) to compute how many tokens appear before each position and where the most recent start occurred. Using the last start information, `map_tokens` finds the token ends and bounds a token to be all characters in between a token end and its corresponding last start. The final output is a vector of sliced tokens, in their input order, with function `f` applied on each. 

We also ported the `block_delayed_scan` function into Rust. It performs parallel prefix sum [19] by dividing the input vector into fixed-size blocks, performs sequential accumulation within each block, and sums up the results of each block for the final prefix sum. The scan is exclusive, only counting up to and excluding the current element.

The `histogram_by_key` function is already ported by the original authors of RPB, which we took advantage of conveniently. It has the equivalent functionality of using a hash map to increment the occurrence counter in the sequential version. This function takes in a vector of tokens generated from the above functions and a hash function. It returns a vector of tuples, where the keys are the unique tokens and the values are the number of occurrences of each token in the input. 

The ported parallel version is a combination of histogram and histogramStar in the original PBBS word count benchmark. We studied the original two benchmarks and concluded that the differences are negligible when porting into Rust. Thus, we decided to merge them into a single benchmark.

The first difference is the data type of the token vector. histogramStar directly calls `map_tokens`, returning a vector of the words. The histogram calls the function `tokens`, wraps each word into a `short_sequence` of characters, and outputs a vector of `short_sequence`s. `sequence` and its variations (e.g., `short_sequence`) are ParlayLib’s special data structure representing parallelizable versions of std::vector [18]. The original RPB authors did not port the sequence data type, and used Rust vectors with Rayon parallel iterators as replacement instead. We choose to implement the same idea by directly returning the words in a vector.

The second difference is the hash function used by `histogram_by_key`. histogram uses ParlayLib’s default hash function, a multiplicative approach, while histogramStar uses djb2 hash function. We choose to use the djb2 hash function to match with the sequential version. 

#### 3.1.2 Inverted Index

Inverted index first breaks the input text into documents (identified by starting string `<doc`), then for each unique word, list the document IDs it appeared in. The output words appear in lexicographical order. Similar to the word count benchmark, all non-alphabetical characters will be replaced as spaces and treated as word delimiters.

**Sequential**

In the sequential version, the first step is to locate the start and end of a document, bound by a pair of the keyword `<doc`. PBBS used the C++ std::search to find the first occurrence of the keyword `<doc` in a given slice of input characters. Then, for each document, we iterate through each character to slice out the tokens, and store the word tokens along with the document they appeared in into a hashmap. The key is the word token and the value is a list of document IDs. We then collect all keys into a separate vector to sort them in dictionary order, and remap sorted keys with the document IDs in the hash map.

**Parallel**

Instead of processing documents one by one, we process all documents in parallel using Rayon’s parallel iterator and call function `tokens` to split words within each document. This generates a vector of token + document ID pairs per document. We ported a new ParlayLib function named `group_by_key` to group separate results from different documents together. Given the token + doc ID pairs, `group_by_key` will flatten them into a single list, group by token, and aggregate the document IDs in a list. Finally, the result is sorted in dictionary order using Rayon’s `par_sort_unstable_by_key`.

We replaced some ParlayLib functions used in the original PBBS with implementations having equivalent functionality. This is due to the Rust-only limitations on some ParlayLib functions introduced by the original RPB authors. This added slight overhead, but we tried not to reimplement these functions to avoid breaking existing RPB tests. 

### 3.2 Computational Geometry

## 4. User’s Guide

To run each ported benchmark, use the following command where `<name>` is the name of the benchmark’s binary. The binary names are listed in `rpb/pbbs/Cargo.toml`.
```
cargo run --release  --bin="<name>" -- <ARGS>
```

Every binary has a help message viewable with
```
cargo run --release  --bin="<name>" -- <ARGS>
```

Every benchmark requires an input file. A sample input file has been provided for all the benchmarks we added. Below is an example command to benchmark every algorithm we have written using its sample input.
```
cargo run --release --bin=hull -- ../test-inputs/hull
cargo run --release --bin=knn -- ../test-inputs/knn --dimension 3 --algorithm naive
cargo run --release --bin=knn -- ../test-inputs/knn --dimension 3 --algorithm chan05
cargo run --release --bin=wc -- ../test-inputs/wc.txt -o=output.txt
cargo run --release --bin=wc -- ../test-inputs/wc.txt -o=output.txt -a=histogram
cargo run --release --bin=index -- ../test-inputs/index.txt -o=output.txt
cargo run --release --bin=index -- ../test-inputs/index.txt -o=output.txt -a=parallel
```

A user can provide custom inputs by modifying the provided files. The files are structured in a fairly intuitive way. To see the precise input specification for any benchmark, see the PBBS benchmark list [S].

## 5. Reproducibility Guide

> [!NOTE]
> This project has been only tested on Linux environment. There is no reason that it will fail on MacOS but the developers have never tested on MacOS.

To run the Rust Parallel Benchmarks suite (RPB), first make sure Rust is installed on your machine. Follow instructions at https://rust-lang.org/tools/install/ to install Rust. Then run the following commands:
```
git clone https://github.com/angelayuxh/ece1724-rusty-benchmarks.git --recurse-submodules
cd ece1724-rusty-benchmarks/rpb
cargo build --release
```

All the required crates are already included in Cargo.toml. Cargo build will automatically download the crates.

PBBS is not required to run our project. However, to compare the performance results of it with RPB, it can be set up by following its installation instructions [U]. PBBS also contains scripts to generate random input files.

## 6. Contributions

Below is the technical  contribution breakdown:

Angela
- Word Count (sequential, parallel)
- Inverted Index (sequential, parallel)

Parisa
- Nearest Neighbors (sequential, parallel)
- Convex Hull (sequential)

## References

1. cmuparlay, “GitHub - cmuparlay/pbbsbench: New version of pbbs benchmarks,” GitHub, 2019. https://github.com/cmuparlay/pbbsbench
2. D. Anderson, G. E. Blelloch, L. Dhulipala, M. Dobson, and Y. Sun, ‘The problem-based benchmark suite (PBBS), V2’, in Proceedings of the 27th ACM SIGPLAN Symposium on Principles and Practice of Parallel Programming, Seoul, Republic of Korea, 2022, pp. 445–447.
3. “rayon-rs/rayon,” GitHub, May 04, 2024. https://github.com/rayon-rs/rayon
4. NASA. “NAS Parallel Benchmarks.” NASA Advanced Supercomputing (NAS) Division. https://www.nas.nasa.gov/software/npb.html accessed October 5th, 2025
5. C. Bienia, S. Kumar, J. P. Singh, and K. Li, ‘The PARSEC benchmark suite: characterization and architectural implications’, in Proceedings of the 17th International Conference on Parallel Architectures and Compilation Techniques, Toronto, Ontario, Canada, 2008, pp. 72–81.
6. bamos, “GitHub - bamos/parsec-benchmark: An unofficial mirror of the core PARSEC 3.0 benchmark suite with patches to run on x86_64 Arch Linux and generalize builds.,” GitHub, 2025. https://github.com/bamos/parsec-benchmark 
7. R. Laso, D. Krupitza, and S. Hunold, ‘pSTL-Bench: A Micro-Benchmark Suite for Assessing Scalability of C++ Parallel STL Implementations’, arXiv [cs.DC]. 2024.
8. https://github.com/parlab-tuwien/pSTL-Bench 
9. NERSC, “C++ parallel algorithms benchmark - NERSC Documentation,” Nersc.gov, 2025. https://docs.nersc.gov/development/programming-models/iso-cpp-parstl-benchmark/cppParSTL/ 
10. weilewei, “GitHub - weilewei/parSTL: Quick C++ Parallel Benchmarks with Different Parallel Frameworks (HPX, Kokkos, GNU, NVhpc, Taskflow),” GitHub, 2023. https://github.com/weilewei/parSTL
11. J. Abdi, G. Zhang, and M. C. Jeffrey, ‘Brief Announcement: Is the Problem-Based Benchmark Suite Fearless with Rust?’, in Proceedings of the 35th ACM Symposium on Parallelism in Algorithms and Architectures, Orlando, FL, USA, 2023, pp. 303–305. 
12. mcj-group, “GitHub - mcj-group/rpb: RPB: Rust Parallel Benchmarks suite,” GitHub, 2024. https://github.com/mcj-group/rpb
13. E. M. Martins, L. G. Faé, R. B. Hoffmann, L. S. Bianchessi, and D. Griebler, ‘NPB-Rust: NAS Parallel Benchmarks in Rust’, arXiv [cs.DC]. 2025.
14. GMAP, “GitHub - GMAP/NPB-Rust: NPB-Rust: NAS Parallel Benchmarks in Rust,” GitHub, 2025. https://github.com/GMAP/NPB-Rust
15. R. Pieper, J. Löff, R. B. Hoffmann, D. Griebler, and L. G. Fernandes, ‘High-level and efficient structured stream parallelism for rust on multi-cores’, Journal of Computer Languages, vol. 65, p. 101054, 2021.
16. GMAP, “GitHub - GMAP/RustStreamBench: Benchmark suite for stream processing with Rust, including versions using parallel APIs such as Rust-SSP, Rayon, Tokio, STD Threads, and Pipeliner.,” GitHub, Aug. 21, 2021. https://github.com/GMAP/RustStreamBench 
17. J. Abdi, G. Posluns, G. Zhang, B. Wang, and M. C. Jeffrey, ‘When Is Parallelism Fearless and Zero-Cost with Rust?’, in Proceedings of the 36th ACM Symposium on Parallelism in Algorithms and Architectures, Nantes, France, 2024, pp. 27–40.
18. G. E. Blelloch, D. Anderson, and L. Dhulipala, ‘ParlayLib - A Toolkit for Parallel Algorithms on Shared-Memory Multicore Machines’, in Proceedings of the 32nd ACM Symposium on Parallelism in Algorithms and Architectures, Virtual Event, USA, 2020, pp. 507–509.
19. Wikipedia Contributors, “Prefix sum,” Wikipedia, Aug. 14, 2025.