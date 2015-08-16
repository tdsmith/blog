---
layout: post
status: publish
title: SDSC Summer Institute Wrapup
---

![Tim and Gordon](/img/supercomputer_selfie.jpeg){: style="float: right"} What would you do with 2 petaflops of processing power? It's not an idle question: NSF's [XSEDE infrastructure](https://www.xsede.org/) offers researchers access to large-scale computing infrastructure. I spent the last week at the [San Diego Supercomputer Center](https://www.sdsc.edu), which hosts an annual Summer Institute to introduce scientists from many disciplines to high-performance computing technologies. Tissue engineering doesn't have a lot of petascale problems but I did learn a lot about tools that can help me write more performant code on single and multi-processor systems—even in Python, my favorite programming language. ~And~ I got to take a selfie with a supercomputer.

## Faster Python code

We touched on a couple of technologies that can make code faster even on a single core on my laptop. [Numba](http://numba.pydata.org/) is a neat library that provides just-in-time compilation of regular ol' numpy-using Python code using LLVM; all you have to do is label your function with a `@jit` decorator. It's a great way to speed up code that can't take advantage of numpy's fast vectorization.

Here's a quick demo -- note that the `slow` and `jitted` implementations are exactly the same, except for the `@jit` decorator:

<blockquote>
{% highlight python %}
#!/usr/bin/env ipython
from numba import jit
import numpy as np

def slow(x):
    sum = 0
    for i in xrange(len(x)-1):
        sum += x[i] * x[i+1]
    return sum

def vectorized(x):
    return (x[1:] * x[0:-1]).sum()

@jit
def jitted(x):
    sum = 0
    for i in xrange(len(x)-1):
        sum += x[i] * x[i+1]
    return sum

x = np.random.random(5000)

%timeit slow(x)
%timeit vectorized(x)
%timeit jitted(x)
{% endhighlight %}
</blockquote>

Here are the results:

Implementation | Time per loop
--- | ---
slow | 1.44 ms
vectorized | 7.79 µs
jitted | 4.83 µs

The vectorized numpy implementation is 70x faster than the naïve slow implementation, but the jitted version of the naïve implementation is even slightly faster than the numpy version, with no extra work!

To use numba you need to have llvm and llvmlite installed; on Homebrew, all you have to do is `brew install homebrew/python/numba`. Continuum Analytics also provides a conda package.

## GPU processing

GPUs offer massive parallelism. The AMD Radeon M370X card in my laptop provides 10 compute units with 64 stream processors each and GPU architectures can efficiently handle multiple threads. Cluster GPU nodes are even more powerful. While tools and interfaces like OpenCL and Nvidia's CUDA have eliminated the need to write GPU code in VHDL, leveraging GPUs fully requires understanding their architecture. In the examples we saw, GPU kernels each perform a small calculation on a single location in memory; the outer layer of a for loop is essentially parallelized over all of the GPU cores, each pushing a result into a single cell of GPU memory. CUDA exposes parameters for allocating jobs to GPU cores and getting them right can have important effects on performance. Main memory management on the device is also handled by the user.

Higher-level APIs like OpenACC, which uses OpenMP-like #pragma syntax, promise to make GPU computing more straightforward but vendor and toolchain support are still works in progress. I'm optimistic about [ArrayFire](https://arrayfire.com), which is a high-level API with Python bindings that abstracts over the GPU's API, so it works with my AMD card on OS X over an OpenCL interface as well as with Nvidia cluster compute nodes.

A straightforward Arrayfire translation of the same problem looks like:

<blockquote>
{% highlight python %}
import arrayfire as af
# use my discrete AMD GPU, not the integrated Isis GPU
af.set_device(1)
def af_vec(x):
    return af.sum(x[:-1] * x[1:])
x = af.randu(500000, 1)
% timeit af_vec(x)
{% endhighlight %}
</blockquote>

Running on the GPU is slower than the numba implementation version for n < about 500,000, but is faster after that. I'll look forward to learning which problems can be solved faster on a GPU and how to implement them optimally.

## Scaling up, scaling out

I attended a session that touched on OpenMP and MPI, which I've always gotten confused before. OpenMP is a classic solution for multi-threaded __shared-memory__ computation on a single node. MPI lets you run multiple __independent processes__ with independent memory spaces on one to _n_ nodes, which can communicate and synchronize with each other through the MPI API. They are often used together. (For the record: OpenMPI is a particular MPI implementation, and has nothing to do with OpenMP!) I don't think either is right for my purposes right now: they're powerful and widely deployed but they work most naturally in lower-level languages like C and Fortran, and it's easy to get burned without a deep appreciation of the challenges and idioms of multiprocessor programming.

While IPython is maybe best known for its notebook view, it also delivers [a powerful set of tools for parallel processing](http://ipython.org/ipython-doc/dev/parallel/parallel_intro.html), expanding well beyond the functionality of the python `multiprocessing` module. IPython has built-in support for spinning up cluster jobs on queue managers like Grid Engine as well as over AWS EC2. I think using IPython's parallel tools will be my first foray into real cluster computing since it provides the infrastructure for moving the single-node multiprocessing I've been doing across a HPC cluster in a straightforward way.

Apache Spark is an interesting tool for friendly parallel computing over large data sets. I was worried about Spark because it's part of the Hadoop ecosystem and my experiences with Hadoop weren't very pleasant but Spark is much easier to use. Spark leverages the Hadoop distributed file system to enable fast, multi-node access to big data and implements a terrifyingly smart abstraction layer over Hadoop's map/reduce architecture to make problem-solving more straightforward and performant, including a suite of ML algorithms. I'll definitely turn to Spark next time I want to deal with machine learning at scale. [Andrea Zonca](https://twitter.com/andreazonca) gave a great talk; his [slide deck](https://github.com/sdsc/sdsc-summer-institute-2015/raw/master/bigdata1_spark/SDSC-SI-Spark.pdf) is worth a perusal if you're even slightly curious.

I should also plug [GNU Parallel](https://www.gnu.org/software/parallel/), which is basically a multiprocessing xargs; it's the easiest way to spin up a job queue of embarrassingly parallel tasks on a single node.

## Resources at SDSC and UCI

SDSC hosts [two XSEDE facilities](https://www.sdsc.edu/services/hpc/hpc_systems.html), both of which are architecturally interesting. Comet embodies the "computing for the 99%" strategy; while it provides 2 Pflop/s over nearly 48,000 computing cores, Comet is optimized for jobs that run on 72 or fewer nodes (1,728 or fewer cores) simultaneously. While some models can only be executed by scaling out to a huge number of nodes, most jobs are much smaller; [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl%27s_law) suggests that many practical problems encounter sharply decreasing returns beyond a certain degree of parallelism. Scheduling an ensemble of these "mid-sized" jobs can be a very effective way to compute. Comet also hosts 36 GPU nodes, each of which boasts 4 Nvidia compute GPUs.

Gordon, the other facility, provides 341 Tflop/s and vast amounts of on-node fast flash storage for data-intensive processing.

Comet, Gordon, and other XSEDE facilities are available through a (somewhat) competitive allocation process, but small grants for exploration and collecting scalability data for competitive grants are available very quickly.

[UC Irvine's facilities](http://hpc.oit.uci.edu/), where I will actually be working, are modest in comparison but still very powerful. Irvine has a small dedicated HPC cluster available to the campus, with 576 cores in 9 nodes available for general use, and another 5,952 cores in 93 condo nodes which are, generously, [available to the community for free](http://hpc.oit.uci.edu/free-queue) when they aren't being used by their owners.

The Summer Institute was a great opportunity to learn from world-class HPC experts and get my hands dirty with some multiprocessing code. If you're curious about anything I touched on, the [slide decks and example code](https://github.com/sdsc/sdsc-summer-institute-2015) from this year's session are available on Github.
