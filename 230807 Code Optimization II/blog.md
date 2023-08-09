# Code Optimization in Scientific Research (Part II)

- [Code Optimization in Scientific Research (Part II)](#code-optimization-in-scientific-research-part-ii)
  - [Introduction and recap](#introduction-and-recap)
  - [The first implementation](#the-first-implementation)
  - [Optimization ideas](#optimization-ideas)
  - [Parallelism](#parallelism)
    - [Core-level](#core-level)
    - [Node-level](#node-level)
    - [Cluster-level](#cluster-level)
    - [Key consideration](#key-consideration)

## Introduction and recap

In [my previous blog](https://labpresse.com/code-optimization-in-scientific-research-part-i/), I introduced some straightforward yet valuable optimization techniques. While these techniques are generally suitable for relatively simple problems, such as the example presented in my previous blog, they may prove inadequate when dealing with more complex and realistic issues. Specifically, in the previous example, my objective was to simulate a single-molecule microscope image. However, people frequently need to process multiple independent images (e.g., frames in a video). In this blog, I will discuss additional techniques within the context of this video simulation problem.

To quickly recap, the previous example involves calculating the total contribution, labeled as $I$, from all molecules. These molecules are indexed by $n$, and they relate to each pixel, which is indexed by $i$ and $j$. Referring to the assumptions discussed earlier, we can express the algorithm's mathematical form as follows: $$I_{ij}=\sum_n \exp[-(x^p_i-x_n)^2-(y^p_j-y_n)^2].$$ Now, shifting our focus to the present issue that involves multiple independent images (or frames), we extend the same calculation to each individual image, denoted as $f$. As a result, the mathematical representation for this new problem takes the following shape (where $V$ stands for video): $$V_{fij}=\sum_n \exp[-(x^p_{fi}-x_{fn})^2-(y^p_{fj}-y_{fn})^2].$$  

## The first implementation

Based on the description so far, we can readily enclose a for-loop iterating over $f$ around the previously optimized code to create the initial version of our single-molecule video simulation code[^1]:

```julia
function video_sim_v1(xᵖ, yᵖ, x, y)
    F = size(x, 3)
    v = Array{eltype(x),3}(undef, length(xᵖ), length(yᵖ), F)
    for f in 1:F
        PSFˣ = exp.(-(xᵖ .- Transpose(view(x, :, 1, f))) .^ 2)
        PSFʸ = exp.(-(view(y, :, 1, f) .- Transpose(yᵖ)) .^ 2)
        v[:, :, f] = PSFˣ * PSFʸ
    end
    return v
end
```

[^1]: `view(x, :, 1, f)` serves the same purpose as `x[:, 1, f]` but with smaller memory allocation.

Two points to note in the code above:

- `x` and `y` are both arrays of dimensions $N\times1\times F$, where $N$ and $F$ represent the number of molecule and number of frames, respectively.
- It appears that we have made a bold assumption that all frames contain an equal number of molecules. However, this assumption is acceptable since molecules that should not appear in a frame can be positioned far away from the field-of-view, thereby making no contribution.

Benchmarking `video_sim_v1` using a dataset comprising 20 molecules and 100 frames (each with 256$\times$256 pixels) yields `50.927 ms (1402 allocations: 123.47 MiB)`. Our overarching goal entails improving upon this benchmark.

## Optimization ideas

Before exploring new techniques, let's take a moment to consider whether we can apply anything from [my previous blog](https://labpresse.com/code-optimization-in-scientific-research-part-i/). Since we have only added one extra loop, there isn't much opportunity to reduce memory allocation. What's more, this extra loop cannot be easily eliminated through vectorization, as the formula specified here doesn't align with basic matrix (or tensor) operations. Consequently, we must use other techniques to tackle this challenge.

In this video simulation problem, it is important to note that all frames are independent of each other. As a result, there is potential to simulate frames simultaneously, or in other words, in parallel.

## Parallelism

Parallelizing an algorithm is much easier said than done. In view of the intricate nature of contemporary computational infrastructures, attaining parallelism in the present era involves three major tiers: core-level parallelism, node-level parallelism, and cluster-level parallelism[^2]. In the upcoming sections, I will delve into a single common scheme within each tier and examine its relevance within the context of our specific problem.

[^2]: Please note that these concepts are not mutually exclusive.

<!-- For each level, I pick one common scheme and demonstrate how my problem is tackled -->


<!-- First, due to the complexity of modern computing facilities, parallelism nowadays can be achieved on roughly three levels: on a core, on a node, and on a cluster. -->

<!-- , parallelism itself actually contains several distinct yet interconnected ideas.  -->


<!-- While different communities may employ various terms, there are approximately three levels of parallelization schemes:  -->


<!-- [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data), [multithreading](https://en.wikipedia.org/wiki/Multithreading_(computer_architecture)), and [multiprocessing](https://en.wikipedia.org/wiki/Multiprocessing). In the subsequent sections, I will discuss and demonstrate each of these methods. -->

<!-- [^1]: Navigating the realm of parallelization schemes can be likened to delving into a rabbit hole :rabbit:, especially for individuals without an expert background. These schemes manifest at different hierarchical levels and, in my personal perspective, often sport comparable and bewildering nomenclature. Within this context, I have chosen to spotlight three common types for "everyday" users. -->

### Core-level

This initial question that may arise is: how is it possible to achieve parallelism on a single core? To illustrate this, let's consider a situation where a program operates on 64-bit integers, and a processor core possesses the capability to fetch 256 bits of data in a solitary operation. In such a scenario, it becomes viable to load four integers as a vector and perform a singular vectorized iteration of the original operation. This could potentially yield a theoretical speedup of fourfold[^3]. This particular approach to parallelization is commonly known as "[single instruction, multiple data](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data)" (SIMD).

<!-- is it possible for a single core to run an algorithm in parallel. One example is to consider a scenario where a program works with 64-bit integers, and a processor core has the capability to load 256 bits of data in a single operation. In this scenario, it becomes feasible to load four integers as a vector and execute a single vectorized version of the original operation, potentially resulting in a theoretical speedup of four times. This specific parallelization scheme is often referred to as "[single instruction, multiple data](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data)", or just SIMD for short. -->

<!-- SIMD is an acronym that stands for "single instruction multiple data". While intricate specifics concerning the implementation of SIMD can be explored in other sources, its fundamental concept remains remarkably straightforward. Consider a scenario where a program works with 64-bit integers, and a processor core has the capability to load 256 bits of data in a single operation. In this scenario, it becomes feasible to load four integers as a vector and execute a single vectorized version of the original operation, potentially resulting in a theoretical speedup of four times[^2]. -->

[^3]: You should now recognize that SIMD is closely related to vectorization (introduced in [my previous blog](https://labpresse.com/code-optimization-in-scientific-research-part-i/#vectorization)). In fact, vectorization constitutes a specific implementation of SIMD principles.

The straightforward concept of SIMD, on one hand, allows numerous modern programming languages to identify points within an algorithm where SIMD can be employed and subsequently apply it automatically. On the other hand, SIMD is frequently constrained to basic operations such as addition or multiplication. Hence, the potential enhancement of `video_sim_v1` through this method remains uncertain. Nevertheless, in this scenario, an attempt must be made to explore the possibilities.

In [Julia](https://julialang.org/), it is possible to enforce vectorization by employing the `@simd` macro, placed before a for-loop involving independent iterations. This technique results in the creation of `video_sim_v2`:

```julia
function video_sim_v2(xᵖ, yᵖ, x, y)
    F = size(x, 3)
    v = Array{eltype(x),3}(undef, length(xᵖ), length(yᵖ), F)
    @simd for f in 1:F
        PSFˣ = exp.(-(xᵖ .- Transpose(view(x, :, 1, f))) .^ 2)
        PSFʸ = exp.(-(view(y, :, 1, f) .- Transpose(yᵖ)) .^ 2)
        v[:, :, f] = PSFˣ * PSFʸ
    end
    return v
end
```

A benchmark analysis yields a result of `51.017 ms (1402 allocations: 123.47 MiB)`, indicating a lack of performance improvement. It seems that Julia has indeed automatically vectorized the code in this case.

### Node-level

Moving up a level there is parallelism on a node (often a computer), which is often achieved through [multithreading](https://en.wikipedia.org/wiki/Multithreading_(computer_architecture)). For multithreading, we require multiple processors (either physical or virtual[^4]), with each core being associated with a separate thread. Multithreading facilitates the simultaneous execution of these processors, all while utilizing the same memory pool. It is important to note that the implementation of multithreading demands careful consideration to avoid conflicts between threads. Fortunately, developers have often shouldered much of this responsibility, alleviating users of this burden.

[^4]: For example, see [hyper-threading](https://en.wikipedia.org/wiki/Hyper-threading)

In Julia, multithreading a for-loop can be as easy as follows[^5]:
[^5]: In order to enable multithreading, certain programming languages may require additional parameters during startup. You can find instructions on how to accomplish this in Julia on [this page](https://docs.julialang.org/en/v1/manual/multi-threading/) shows how to do it in Julia.

```julia
function video_sim_v3(xᵖ, yᵖ, x, y)
    F = size(x, 3)
    v = Array{eltype(x),3}(undef, length(xᵖ), length(yᵖ), F)
    Threads.@threads for f in 1:F
        PSFˣ = exp.(-(xᵖ .- Transpose(view(x, :, 1, f))) .^ 2)
        PSFʸ = exp.(-(view(y, :, 1, f) .- Transpose(yᵖ)) .^ 2)
        v[:, :, f] = PSFˣ * PSFʸ
    end
    return v
end
```

My desktop computer is equipped with four physical CPU cores, which translate into eight threads. When assessing the benchmark of `video_sim_v3` with all eight threads, the results demonstrate a remarkable speedup of almost four times comparing to `video_sim_v1`, clocking in at `12.925 ms (1450 allocations: 123.47 MiB)`.

### Cluster-level

Now assume you have access to a cluster, which is not uncommon for universities and institutes nowadays, you could even consider modifying the algorithm to execute across multiple processors spanning numerous computers. A frequently employed strategy involves the utilization of [multiprocessing](https://en.wikipedia.org/wiki/Multiprocessing).

With the concept of multithreading in mind, we can easily comprehend multiprocessing as the simultaneous operation of multiple processors, where each core has access only to its designated memory space. This fundamental distinction from multithreading requires some "coding maneuvers" as users are now compelled to determine the allocation of data to individual processors. In the context of our example problem, implementing multiprocessing requires some rather major change of the code, contradicting the very impetus driving my blog posts. Therefore, I only provide a preliminary example in [this GitHub repository of mine]().

### Key consideration

<p align="center" height="100%">
    <img src="fig1.png">
</p>

While I aimed to maintain a surface-level discourse in my blog, it is totally reasonable to feel confused when deciding upon a parallelization scheme. :smile: The crucial factor to bear in mind is that an escalation in the number of processors engaged directly corresponds to an increase in communication costs. This rise in costs can potentially overshadow the performance benefits gained from task distribution.