## the setup
I chose to benchmark the dot product operation because it’s a cheap operation, constrained only by the input size.
The input will consist of two float vectors, _a_ and _b_, of size _n_.
<figure>
	<img class="inverted-svg" src="/images/dot-product.svg">
	<figcaption>Where <i>a</i> and <i>b</i> are vectors, <i>n</i> is the size of the vectors, and <i>ai</i> & <i>bi</i> are vector components belonging to vectors <i>a</i> & <i>b</i>.</figcaption>
</figure>

The input size is computed as _n_ **x** float size (bytes) **x** number of vectors. To avoid memory shortage, I capped the input size at 256 MB.

I ran all my benchmarks on a Quad core Intel Core i7-8550U CPU with a 1MB L2 cache and an 8MB L3 cache. This CPU supports AVX2.

## naive scalar function
The first step is to establish a baseline benchmark. Here's a naive implementation of the dot product function: 


<script src="https://gist.github.com/ishuah/bf48151ac9a9709d5a86ce1ac5800233.js"></script>

I comp
<figure>
    <img src="/images/dot_product_naive_benchmark.png">
    <figcaption>Bash runs as a subprocess, spawned by our program. Bash receives the command `ping 8.8.8.8` and spawns a subprocess.</figcaption>
</figure>

#### AVX2 function

#### AVX2 function with two accumulators
