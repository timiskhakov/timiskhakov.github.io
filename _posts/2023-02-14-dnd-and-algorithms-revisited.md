---
layout: post
title: D&D and Algorithms Revisited
excerpt: Rolling the dice using fast computation of binomial coefficients and caching
tags: [go, algorithms, performance]
---

Last time we [were rolling](dnd-and-algorithms) the dice and calculating probabilities of getting certain sums, we used approaches that relied on brute-forces and binomial coefficients. We got some interesting results and verified that using a deterministic formula based on computing binomial coefficients was, in fact, more efficient than going through the hassle of enumerating all possible die combinations. In this post, we are going to advance both approaches and see if some clever tricks could make the brute-force more practical.

# Fast Computation of Binomial Coefficients

Let's start with the approach that employed [binomial coefficients](https://en.wikipedia.org/wiki/Binomial_coefficient). I'm not going to list the full implementation from the previos post, but let's just briefly recap that the core idea lied in computing a binomial coefficient using the following function:

```go
func coef(n, k int) int {
  c := 1
  for i := 0; i < min(k, n-k); i++ {
    c *= (n - i) / (i + 1)
  }
  return c
}

func min(n, k int) int {
  if n < k {
    return n
  }
  return k
}
```

Normally, a binomial coefficient is computed using factorials:

$$ \dbinom n k = \dfrac{n!}{k!(n - k)!} $$

However, since factorials can easily overflow the size of `int`, we switched to building up the coefficient by calculating intermediate results.

This is definitely not the most optimal approach of calculating a binomial coefficient, in no small part due to division that's located in the loop. While division might not seem like a complex operation for humans, it actually is for computers. From a hardware perspective, division is an iterative algorithm, the duration of which is directly proportional to the number of bits involved. In performance-critical code, division often gets replaced by a combination of cheaper operations, such as addition, multiplication, and shift to speed up calculations.

Since we're writing performance-critical code here — just for fun, of course, but still — we'll perform an optimization which removes division from our function using an approach described by Daniel Lemire in his article [Fast divisionless computation of binomial coefficients](https://lemire.me/blog/2020/02/26/fast-divisionless-computation-of-binomial-coefficients/). The idea of the approach is to precompute a series of pairs of numbers for replacing division with a combination of shift and multiplication[^1]. We're not going to invent the wheel, rather we'll just port Lemire's [implementation](https://github.com/lemire/Code-used-on-Daniel-Lemire-s-blog/blob/master/2020/02/26/binom.c) from C to Go.

First, since we know that the maximum value of `k` is limited due to the problem's constraints, we can precompute 64 pairs like that:

```go
type fastdiv struct {
  shift   int
  inverse uint64
}

var precomputed = []fastdiv{
  {0, 0},
  {0, 0x1},
  {1, 0x1},
  {0, 0xaaaaaaaaaaaaaaab},
  {2, 0x1},
  {0, 0xcccccccccccccccd},
  {1, 0xaaaaaaaaaaaaaaab},
  {0, 0x6db6db6db6db6db7},
  // ...
  {0, 0xefbefbefbefbefbf},
}
```

Then we can create a new function for computing binomial coefficients that's using precomputed data:

```go
func fastcoef(n, k int) int {
  if k == 0 {
    return 1
  }

  np := n - k
  c := np + 1
  for i := 2; i <= k; i++ {
    c *= np + i
    f := precomputed[i]
    c >>= f.shift
    c = int(uint64(c) * f.inverse)
  }

  return c
}
```

This is a drop-in replacement for our `coef` function from the previos post. Now if we change methods in the implementation and run our small benchmark that rolls a 6-sided die 10 times and counts sums of 31, we can see that we surely gained some boost in performance:

```
goos: linux
goarch: amd64
pkg: github.com/timiskhakov/dnd/src
cpu: Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz
BenchmarkNaive-12                      2         596836789 ns/op
BenchmarkSmart-12                     52          21166725 ns/op
BenchmarkBinomial-12             3334281             345.5 ns/op
BenchmarkLemire-12              12011446              90.5 ns/op
```

Bare in mind, though, I'm running this on a x64 processor while employing a quite specific hardware trick. Other chips, like Apple's M1 and M2, may give different results.

# Smart Brute-Force with Caching

When I showed the results to my friend [Alexei](https://github.com/alexeimatrosov) who authored the smart brute-force from the previous post, he immediately came up with an improved version of his implementation that employed a clever trick with caching intermediate computations. Let's quickly have a look at his initial solution:

```go
func Smart(s, n, m int) int {
  if m < n || s*n < m {
    return 0
  }
  if n == 1 {
    return 1
  }

  result := 0
  for i := 1; i <= s; i++ {
    result += Smart(s, n-1, m-i)
  }

  return result
}
```

Simple and elegant, isn't it? Granted, not particularly fast according to the benchmarks above. However, Alexei noted that for unique `s`, `n`, and `m` we could store intermediate results of the function in a cache to re-use them earlier in the function. So instead of falling into the recursion every time, we just use cached results we already calculated:

```go
type item struct {
  s, n, m int
}

var cache = make(map[item]int)

func Cache(s, n, m int) int {
  if m < n || s*n < m {
    return 0
  }
  if n == 1 {
    return 1
  }

  i, ok := cache[item{s, n, m}]
  if ok {
    return i
  }

  result := 0
  for i := 1; i <= s; i++ {
    result += Cache(s, n-1, m-i)
  }

  cache[item{s, n, m}] = result

  return result
}
```

By applying this simple optimization, that is not tied to any specific platform, by the way, we squeeze even more performance:

```
oos: linux
goarch: amd64
pkg: github.com/timiskhakov/dnd/src
cpu: Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz
BenchmarkNaive-12                      2         548191400 ns/op
BenchmarkSmart-12                     60          19837065 ns/op
BenchmarkBinomial-12             3366622             358.5 ns/op
BenchmarkLemire-12              13037398              87.8 ns/op
BenchmarkCache-12               49282900              21.6 ns/op
```

Well, what can I add? Bravo, Alexei! You can check out code from this post in the same repository: [dnd](http://github.com/timiskhakov/dnd).

# Footnotes

[^1]: This fact is descibed in detail in Lemire's paper [Faster Remainder by Direct Computation: Applications to Compilers and Software Libraries](https://arxiv.org/abs/1902.01961)