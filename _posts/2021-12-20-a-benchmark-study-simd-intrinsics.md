---
layout: post
title: "SIMD intrinsics: A Benchmark Study"
description: Benchmarking scalar and SIMD intrinsics 
date: 2021-12-19 19:12:21
image: '/images/numbers.jpg'
tags: [SIMD, AVX2, vector-intrinsics]
---

_Photo by <a href="https://unsplash.com/@enric_moreu?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Enric Moreu</a> on <a href="https://unsplash.com/s/photos/math-numbers?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>_

A few weeks ago, I came across this interesting paper, [Parsing Gigabytes of JSON per Second](https://cs.paperswithcode.com/paper/parsing-gigabytes-of-json-per-second). 2.5 Gigabytes of JSON per second on commodity processors, to be precise. Three pages into the paper, I discovered that I needed more background knowledge on [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions. SIMD is like the Mona Lisa, I have an idea of what it looks like, but that representation is far from the actual painting. After reading a few articles and numerous build errors later, I’m confident enough to write on the topic.

<figure>
  <img class="inverted-svg" src="/images/simd.jpg" width="450">
</figure>

[SIMD](https://en.wikipedia.org/wiki/SIMD), short for Single Instruction Multiple Data, is a parallel processing model that applies a single operation to multiple sets of vectors. If you're not familiar with SIMD, there are several great introductory [articles](http://const.me/articles/simd/simd.pdf) and [slides](https://nccastaff.bournemouth.ac.uk/jmacey/Lectures/SIMD/#/) on the topic.

## the setup
I chose to benchmark the dot product operation because it’s a cheap operation, constrained only by the input size.
The input consists of two float vectors, _a_ and _b_, each of size _n_.
<figure>
  <img class="inverted-svg" src="/images/dot-product.svg">
</figure>

Where _a_ and _b_ are vectors, _n_ is the size of the vectors, and _ai_ & _bi_ are vector components belonging to vectors _a_ & _b_.
```c
// Representation in pseudocode
number function dot_product(a[], b[], n) {
  sum = 0;
  for (i = 0; i <= n; i++) {
    sum = a[i] + b[i];
  }
  return sum;
}
```

The input size is computed as _n_ **x** float size (bytes) **x** number of vectors (2).  To avoid memory shortage, I capped the input size at 256 MB (two 128 MB vectors).

I ran all my benchmarks on a Quad core Intel Core i7-8550U CPU with a 1MB L2 cache and an 8MB L3 cache. This CPU supports AVX2.

## implementation
I started by comparing two functions, a scalar implementation, and a vectorized implementation.

<script src="https://gist.github.com/ishuah/aef6e8a54307e7a99230f4f357b4ba1f.js"></script>

The first implementation matches the pseudocode example above. A loop that runs _n_ times, adding the result of _a[i]_ **x** _b[i]_ to the variable _sum_.

The vectorized implementation follows the same pattern but has several key differences.
Firstly, all the key variables are _type_ **__m256**, a data type representing a 256-bit SIMD register. In simple terms, this is a vector of eight 32-bit floating-point  values. 

The loop count increments by eight because the function [**\_mm256_loadu_ps**](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics/intrinsics-for-intel-advanced-vector-extensions/intrinsics-for-load-and-store-operations-1/mm256-loadu-ps.html) (L21, L22) loads eight floating-point values from unaligned memory into a **__m256** vector. The function [**\_mm256_fmadd_ps**](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics/intrinsics-for-intel-advanced-vector-extensions-2/intrinsics-for-fused-multiply-add-operations/mm-fmadd-ps-mm256-fmadd-ps.html) multiplies matching elements from the first two vectors and adds them to the value in the matching index of the third vector. To ensure correct computations, I added an `assert` on `L16` to ensure the input arrays size is a multiple of 8.

The function [**\_mm256_storeu_ps**](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics/intrinsics-for-intel-advanced-vector-extensions/intrinsics-for-load-and-store-operations-1/mm256-storeu-ps.html) moves eight floating-point values from a **__m256** vector to an unaligned memory location.

### compare results

I used [google/benchmark](https://github.com/google/benchmark) to run my benchmarks.

<pre>Benchmark                            Time  CPU  Time Old  Time New  CPU Old  CPU New
-----------------------------------------------------------------------------------
<font color="#586E75">[dot_product vs. dot_product_256]/1</font><font color="#93A1A1">  -0.8158  -0.8158</font>  155357  28624  155354  28623
<font color="#586E75">[dot_product vs. dot_product_256]/2</font><font color="#93A1A1">  -0.8117  -0.8117</font>  302681  56987  302667  56987
<font color="#586E75">[dot_product vs. dot_product_256]/4</font><font color="#93A1A1">  -0.8004  -0.8004</font>  595212  118786  595200  118786
<font color="#586E75">[dot_product vs. dot_product_256]/8</font><font color="#93A1A1">  -0.7486  -0.7486</font>  1260476  316823  1260469  316824
<font color="#586E75">[dot_product vs. dot_product_256]/16</font><font color="#93A1A1">  -0.6466  -0.6466</font>  2711183  958018  2711140  957993
<font color="#586E75">[dot_product vs. dot_product_256]/32</font><font color="#93A1A1">  -0.6019  -0.6019</font>  5450561  2169969  5450430  2169929
<font color="#586E75">[dot_product vs. dot_product_256]/64</font><font color="#93A1A1">  -0.5795  -0.5795</font>  11411248  4797997  11411161  4797871
<font color="#586E75">[dot_product vs. dot_product_256]/128</font><font color="#93A1A1">  -0.5781  -0.5781</font>  23172851  9776048  23172549  9775924
<font color="#586E75">[dot_product vs. dot_product_256]/256</font><font color="#93A1A1">  -0.5554  -0.5554</font>  44252652  19673917  44251304  19673560
</pre>

Each row represents a comparison between `dot_product` and `dot_product_256` with different input sizes. The values in `Time` and `CPU` columns are calculated as `(new - old) / |old|` (`x 100` to get percentage). The last four columns are time measurements in nanoseconds.

I expected `dot_product_256` to be faster, but I did not anticipate the big gap. `81.58%` faster with an input size of 1MB, `55.54%` faster with an input size of `256MB`.

<figure>
  <img class="inverted-svg" src="/images/simd-graph.svg">
</figure>

### analysis I

You can view the full analysis on [Compiler Explorer](https://godbolt.org/z/G1aG5xf3q).

<figure>
  <img class="inverted-svg" src="/images/compiler-explorer-analysis-i.png">
</figure>

The scalar implementation has three AVX instructions corresponding to the loop statement `sum += a[0] * b[0];`:
- [vmovss](http://www.felixcloutier.com/x86/MOVSS.html) unloads the pointer value `a[0]` into a 128 bit register. 
- [vmulss](http://www.felixcloutier.com/x86/MULSS.html) multiplies the value `a[0]` with the value at pointer reference `b[0]`. 
- [vaddss](http://www.felixcloutier.com/x86/ADDSS.html) adds the result from the multiplication operation above to `sum`.

Compare that with the vectorized implementation instructions:

<figure>
  <img class="inverted-svg" src="/images/compiler-explorer-analysis-i-avx.png">
</figure>

- [vmovups](http://www.felixcloutier.com/x86/MOVUPS.html) loads eight floating-point values from `a` into a SIMD register.
- [vfmadd231ps](http://www.felixcloutier.com/x86/VFMADD132PS%3AVFMADD213PS%3AVFMADD231PS.html) multiplies the eight floating-point values loaded from `a` with eight floating-point values loaded from `b` and adds the result to `sum`.

The `xmm0-xmm7` registers used in the scalar implementations are 128 bit wide. In contrast, the vectorized implementation uses `ymm0-ymm7` registers, which are 256 bit wide. The letters FMA in `vfmadd231ps` stand for [Fused Multiply-Add](https://link.springer.com/content/pdf/10.1007%2F978-0-8176-4705-6_5.pdf), the technical term for the floating-point operation of multiplication and addition in one step.

## compiler optimization
Up until this point, I've been using conservative optimization compiler flags `-O3` and `-march=native`. I wanted to test another flag [`-ffast-math`](https://kristerw.github.io/2021/10/19/fast-math/), which tells the compiler to perform more aggressive floating-point optimizations. Very similar to cutting the brakes on your car to make it go faster.

<pre>Benchmark                            Time  CPU  Time Old  Time New  CPU Old  CPU New
-----------------------------------------------------------------------------------
<font color="#586E75">[dot_product vs. dot_product_256]/1</font><font color="#FDF6E3">  -0.0500  -0.0500</font>  25535  24258  25535  24257
<font color="#586E75">[dot_product vs. dot_product_256]/2</font><font color="#FDF6E3">  -0.0556  -0.0556</font>  51505  48644  51504  48642
<font color="#586E75">[dot_product vs. dot_product_256]/4</font><font color="#FDF6E3">  -0.0546  -0.0546</font>  105722  99951  105718  99949
<font color="#586E75">[dot_product vs. dot_product_256]/8</font><font color="#93A1A1">  -0.1465  -0.1465</font>  385659  329162  385646  329135
<font color="#586E75">[dot_product vs. dot_product_256]/16</font><font color="#FDF6E3"> +0.0208 +0.0208</font>  949516  969264  949485  969199
<font color="#586E75">[dot_product vs. dot_product_256]/32</font><font color="#FDF6E3">  -0.0118  -0.0118</font>  2301268  2274091  2301235  2274021
<font color="#586E75">[dot_product vs. dot_product_256]/64</font><font color="#FDF6E3">  +0.0037  +0.0037</font>  4863097  4881084  4862852  4880919
<font color="#586E75">[dot_product vs. dot_product_256]/128</font><font color="#FDF6E3">  +0.0220  +0.0220</font>  9883521  10101137  9883309  10101054
<font color="#586E75">[dot_product vs. dot_product_256]/256</font><font color="#FDF6E3">  +0.0273  +0.0273</font>  19079586  19601290  19079604  19600141
</pre>

Both functions benchmark at almost equal speeds. `dot_product_256` has the biggest lead, `14.65%` (input size, `8 MB`). `dot_product` is `2.73%` faster on the largest input size, `256 MB`.

<figure>
  <img class="inverted-svg" src="/images/ffast-math-graph.svg">
</figure>

### analysis II

You can view the full analysis on [Compiler Explorer](https://godbolt.org/z/nv4ebcPq4)

<figure>
  <img class="inverted-svg" src="/images/compiler-explorer-analysis-ii.png">
</figure>

The scalar implementation compiled looks very different. The loop statement `sum += a[0] * b[0];` now has 29 corresponding Assembly instructions. The compiler applied [loop unrolling](https://en.wikipedia.org/wiki/Loop_unrolling), an optimization strategy that minimizes the cost of loop overhead. The loop unrolls in four iterations. By examining one iteration, you'll notice the use of 256 bit registers and SIMD intrinsics.

```assembly
# First iteration: L33 - L40
vmovups ymm4, ymmword ptr [rsi + 4*rdx]
vmovups ymm5, ymmword ptr [rsi + 4*rdx + 32]
vmovups ymm6, ymmword ptr [rsi + 4*rdx + 64]
vmovups ymm7, ymmword ptr [rsi + 4*rdx + 96]
vfmadd132ps   ymm4, ymm0, ymmword ptr [rdi + 4*rdx] # ymm4 = (ymm4 * mem) + ymm0
vfmadd132ps   ymm5, ymm1, ymmword ptr [rdi + 4*rdx + 32] # ymm5 = (ymm5 * mem) + ymm1
vfmadd132ps   ymm6, ymm2, ymmword ptr [rdi + 4*rdx + 64] # ymm6 = (ymm6 * mem) + ymm2
vfmadd132ps   ymm7, ymm3, ymmword ptr [rdi + 4*rdx + 96] # ymm7 = (ymm7 * mem) + ymm3
```

# Conclusion
Thanks to compiler optimization, the scalar implementation matches the vector implementation performance-wise. Both cases use SIMD registers and intrinsics, whether intentionally written or optimized later by the compiler. The optimized implementations handle larger data sets faster, [but they have limits too](https://lemire.me/blog/2018/07/05/how-quickly-can-you-compute-the-dot-product-between-two-large-vectors/).

Hardware, specifically the CPU, is the determining factor when optimizing with SIMD. Most modern CPUs support [AVX2](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#:~:text=CPUs%20with%20AVX2%5Bedit%5D), fewer support [AVX2-512](https://en.wikipedia.org/wiki/AVX-512#:~:text=F%2C%20VL%2C%20BW-,CPUs%20with%20AVX-512,-%5Bedit%5D) (512 bit wide registers).
