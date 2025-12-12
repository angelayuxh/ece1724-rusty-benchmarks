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

The PBBS benchmark suite is built on ParlayLib [18] which is a parallel algorithm library developed in C++. The original authors already ported part of the ParlayLib functions into Rust; we further contributed to the functions that are missing in the original RPB. We ported a total of 4 benchmarks from the text processing and computational geometry categories, along with their required ParlayLib functions.

### 3.1 Text Processing

Two benchmarks under the text processing category are ported, Word Count and Inverted Index. We ported a sequential version and a parallel version for each of them.

#### 3.1.1 Word Count

Word count returns the number of occurrences of each word in the input text file. All non-alphabetical characters will be replaced as spaces and treated as word delimiters. 

**Sequential**

In the sequential version, the input text is chopped into tokens using the native Rust function `split_ascii_whitespace`. Then, we used Rust’s hash map to count the occurrence of each token, where the key is the word/token, and the value is the occurrence counter. Instead of using the native hash function, we replaced it with the djb2 hash used by the original PBBS benchmark to ensure a fair performance comparison. 

**Parallel**

The parallel version of word count replaced the sequential tokenization and writing into hash map with parallelized algorithms from ParlayLib, `tokens` and `histogram_by_key`, respectively. We also used parallel iterators from the Rayon library to enable parallelized access on different data structures whenever possible. 

We ported the function `tokens` and its required sub-functions into Rust. `tokens` calls the function `map_tokens` internally. `map_tokens` performs parallel splitting of the input sequence of characters into words, with the default delimiter being whitespaces, and applies the function `f` on each word/token. In this case, `f` is set to directly return the underlying token. `map_tokens` first marks every token start, then performs a parallel prefix-scan (by calling `block_delayed_scan`) to compute how many tokens appear before each position and where the most recent start occurred. Using the last start information, `map_tokens` finds the token ends and bounds a token to be all characters in between a token end and its corresponding last start. The final output is a vector of sliced tokens, in their input order, with function `f` applied on each. 

We also ported the `block_delayed_scan` function into Rust. It performs parallel prefix sum [19] by dividing the input vector into fixed-size blocks, performs sequential accumulation within each block, and sums up the results of each block for the final prefix sum. The scan is exclusive, only counting up to and excluding the current element.

The `histogram_by_key` function is already ported by the original authors of RPB, which we took advantage of. It has the equivalent functionality of using a hash map to increment the occurrence counter in the sequential version. This function takes in a vector of tokens generated from the above functions and a hash function. It returns a vector of tuples, where the keys are the unique tokens and the values are the number of occurrences of each token in the input. 

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

Two computational geometry benchmarks are ported, nearest neighbours and convex hull, with 2 parallel algorithms for the former and a sequential algorithm for the latter.

### 3.2.1 Nearest Neighbours

The nearest neighbours problem, or knn takes as input a list of points in 2 or 3 dimensions and for each point returns indices of the k nearest points to it. Both algorithms we ported assume k=1.

**Naive**

To find the nearest neighbour of a point, the naive algorithm simply iterates over all the other points and computes the distance to each. We do this for every point in parallel.

**Chan05**

Chan05 is taken from the paper [20]. It begins by converting the points, which are given as double precision floating point numbers, into integers. This is done by normalizing the range of values, so that the smallest value in any dimension always translates to 0. The number of bits used is 20 in the 3d case and 30 in the 2d, which results in a small loss of precision. If two points are nearly equidistant to, we might mistakenly choose the one that’s slightly further. A case-by-case analysis is required to determine if this is an acceptable tradeoff.

In this form, we can bisect the points along the x-axis based on the leading bit in the x-component, and further bisect using lower order bits. This is similarly true for y and z. We organize the points by recursively splitting along rotating axes. For example, for a 2d point set, we’d first bisect the points into two groups along the x-axis, then bisect each group along the y-axis, then the x-axis, and so forth until every point is in its own group. This divides points hierarchically as a tree, known as a quadtree in 2d, or an octree in 3d. The tree is represented implicitly in an array.

For a given reference point, the algorithm now begins by computing its distance to the point in the middle of the array, and updating the nearest neighbour if it’s closer than any previously computed point (which will always be the case the first time). Then it recursively searches the left and right halves of the array. By organizing the points as a quadtree/octree, it may sometimes be the case that a slice of points in the array cannot possibly be closer to the reference point than the current nearest neighbour, allowing us to prune this part of the search, and ultimately making this algorithm much faster than the naive algorithm.

## 3.2.2 Convex hull

The convex hull problem asks us to find the smallest convex polygon that contains a set of points in 2D. The vertices of the polygon will always be points in the set, so the output is a list of indices to the points. Although the problem can be generalized to higher dimensions, doing so greatly increases the complexity.

**Sequential**

Only the sequential algorithm has been ported so far, future work will be necessary for the parallel version.

The algorithm is recursive. First we locate the left- and rightmost points in the set and divide all the points into those above and below the line segment connecting the two. The final solution will be: the leftmost point, a convex hull of the points above the line, the rightmost point, then a convex hull of the points below the line. For the upper portion, find the point above the line that maximizes the area of a triangle with it, and the leftmost and rightmost points as its verticies. This point will be part of the final solution. Divide the remaining points into those to the left and right of the triangle and solve them recursively. We can ignore any points inside the triangle. 


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

## 7. Results

We tested both Rust and C++ on ECF Linux machines. ECF machines have Intel(R) Xeon(R) Gold 6230 CPU @ 2.10GHz, with 80 CPU cores available. We tried our best to test on unloaded machines but results may still be affected by live machine workloads. All results below are the mean of 5 rounds of run.

| Algorithm | Input Description | Rust (seconds) | C++ (seconds) |
| - | - | - | - |
| word count - sequential | 250 million characters | 1.067 |2.721 |
| word count - parallel | 250 million characters | 1.532 | 0.168 |
| inverted index - sequential | 250 million characters | 7.772 | 6.258 |
| inverted index - parallel | 250 million characters | 3.712 | 0.278 |
| nearest neighbours - naive (sequential) | 100 thousand points in 3d | 21.054 | 28.411 |
| nearest neighbours - naive | 100 thousand points in 3d | 0.781 | 0.932 |
| nearest neighbours - chan05 (sequential) | 2 million points in 3d | 3.321 | 9.085 |
| nearest neighbours - chan05 | 2 million points in 3d | 0.131 | 0.203 |
| convex hull - sequential | 100 million points in 2d | 4.880 | 3.923 |

Across the benchmarks, we find consistently worse scaling in Rust with respective speedups of 0.7x, 2.1x, 27.0x, and 25.4x, for word count, inverted index, nearest neighbours, and convex hull whereas in C++ we obtained 16.2x, 22.5x, 30.5x, and 44.8x. Word count was the only benchmark where the sequential algorithm outperformed the parallel version. The sequential version even outperformed its C++ equivalent by nearly 2.6x. We suspect that this is because the native Rust function, `split_ascii_whitespace`, is very efficient, outperforming the C++ approach using `strtok` with a while loop.

On average, Rust sequential implementation is 24% faster than C++. Parallel text processing benchmarks are 11x slower than C++, and nearest neighbors are 19% faster than C++. When we find C++ scaling better than Rust on another problem this could be due to any combination of a strong sequential Rust implementation (as we saw with word count), a poor parallel Rust implementation, a poor C++ sequential implementation, or a strong C++ parallel implementation. Unlike the other 3 problems, the nearest neighbours algorithms don’t have separate sequential and parallel implementations. The sequential results are obtained by running the parallel algorithm on a single core. This can make comparisons easier as there are fewer variables. Focusing just on nearest neighbours, the results we find are still well aligned with the original RPB paper [O], which show Rust outperforming C++ in sequentially, but having worse speedup. This suggests that although Rayon is very easy to use, it probably has some room for optimization.

## 8. Lessons Learned and Concluding Remarks

Although a staple of introductory computer science classes, linked lists rarely see use in practice. In fact, the Rust documentation [T] generally recommends against it saying:

> NOTE: It is almost always better to use Vec or VecDeque because array-based containers are generally faster, more memory efficient, and make better use of CPU cache

The convex hull algorithm seems like it may be an exception to this rule. The algorithm will divide an array of points into two groups, with some unimportant elements in between, then solve for the convex hull over each group. At the end, we want a contiguous solution, so the elements in the right group are copied over to the left. Because the algorithm is recursive, the same elements can be copied many times. If instead we output linked lists, we can take advantage of their O(1) concatenation. Although this offered some speedup, the difference was extremely negligible, so we stuck to using arrays to stay consistent with PBBS in our comparison.

One other takeaway is that the Rayon crate is very powerful and super easy to use, but still needs developers to optimize the code for best performance. Rayon provided a complete set of iterator adapters working with its parallel iterators. Cases with parallel reads can be done by simply replacing the native iterator with Rayon’s parallel iterator, while keeping the rest of the code the same. It also provides a safe interface for lots of concurrent operations that might otherwise need unsafe code. Adapters on parallel iterators like map and filter let the programmer operate in parallel, on collections of data, which is normally unsafe, in a way that’s no riskier than when writing sequential code. However, we noticed some parallel iterator adaptors, such as `filter_map` and `flat_map`, are performance bottlenecks. Manually tweaking the code to avoid these iterator adapters significantly improved runtime.

We noticed some limitations of the ported ParlayLib that were introduced by the original RPB authors that do not exist in the C++ version. For example, the function `remove_duplicates` is restricted to inputs that are primitive integers, thus we cannot use it for processing text inputs. Also, input types of certain functions are restricted to Copy traits, making these functions unusable when processing strings, vectors, etc. We had to relax the Copy traits to Clone traits to match with the functionality in C++ version.

Another limitation we encountered is in convincing the Rust compiler that a set of writes are to distinct indices. In the Chan05 algorithm, we begin by sorting the points, however we want to output the nearest neighbour of each point in the same order as the input. Thus we convert each point to a tuple containing its initial location before sorting so we can relate the results back at the end. For example suppose there are 4 input points [p<sub>0</sub>, p<sub>1</sub>, p<sub>2</sub>, p<sub>3</sub>]. We would augment this to create  [(0, p<sub>0</sub>), (1, p<sub>1</sub>) (2, p<sub>2</sub>), (3, p<sub>3</sub>)], then sorting might yield something like [(3, p3), (0, p<sub>0</sub>), (1, p<sub>1</sub>), (2, p<sub>2</sub>)]. The nearest neighbours are computed in parallel to give [(3, nn<sub>0</sub>), (0, nn<sub>1</sub>), (1, nn<sub>2</sub>,), (2, nn<sub>3</sub>)]. Then we want to write each point into the output array based on the index in the tuple giving [nn<sub>1</sub>, nn<sub>2</sub>, nn<sub>3</sub>, nn<sub>0</sub>]. Attempting to do these writes in parallel, Rust has no way of knowing that they’re safe to do without synchronization. We elected to do this final step sequentially, and due to false sharing [V], this might not be any slower.

In conclusion, the Rust compiler does seem well built for the development of performance critical applications, even compared to C and C++. Furthermore, Rust’s memory management rules, although restrictive at times, are very easy to work within much of the time when using libraries. However, the restriction can still impede development sometimes, and we still aren’t managing to leverage multi-core systems as well as C and C++. Both of these issues can be addressed with improvements to the Rust compiler and library ecosystem.

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
20. Timothy M. Chan, "A Minimalist's Implementation of an Approximate Nearest Neighbor
Algorithm in Fixed Dimensions*", 2006. https://tmc.web.engr.illinois.edu/sss.pdf