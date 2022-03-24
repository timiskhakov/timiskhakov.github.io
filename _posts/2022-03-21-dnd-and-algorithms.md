---
layout: post
title: D&D and Algorithms
excerpt: Rolling the dice using brute-force and binomial coefficients
---

A while ago, a friend of mine had asked me to help with a math problem for a video game that employed D&D-like mechanics. As someone who sunk countless hours into Baldur's Gate and Neverwinter Nights in his teens, I immediately was hooked by the mere mention of D&D. And, of course, I wanted to help my friend solve the issue.

The problem turned out to be fairly simple – on the surface at least. If we have a 6-sided die rolled on the table 3 times, what values do we need to get with each roll to obtain a certain sum and how many combinations could we get in total? That is, if we were looking for a sum of 4, then we'd need the first roll to give us 1, the second one — 1 again, and the third one – 2. Or the first and third rolls to throw 1, but the second one — 2. Or the first and second to give us 1, but the third one — 2. As we see, for the sum of 4 the answer is 3 combinations.

For sums like 3 or 18, the answer is obvious. For 4 or 17, it's perhaps less obvious, but still simple — as we just saw. However, the amount of combinations increases once we go somewhere in the middle, like 8 or 9.

The problem appeared interesting. And what do programmers usually do with a problem they find interesting? Yes, they generalize it.

## Problem

Let's expand this problem by stating that we have a die with the `s` number of sides. We roll it `n` times and we are counting sums equal to `m`. Since we will be solving this problem on an x64 machine that has finite space for storing integer numbers, we would easily overflow calculations if we roll a multi-sided die numerous times. So let's also put the following constraints on our inputs:
- 4 <= `s` <= 10
- 1 <= `n` <= 10
- `n` <= `m` <= `s * n`

## Naive Brute-Force

A solution to the problem might seem simple: all we need to do is to create a state of all possible roll results and check if its sum is equal to `m`. For our initial case with a 6-sided die, 3 rolls, and a sum of 10 it would be:
```
state = [1, 1, 1], sum = 3, m = 10
```

Then we increase roll results one by one and continue checking against `m` until we hit it:

```
state = [1, 1, 2], sum = 4, m = 10
state = [1, 1, 3], sum = 5, m = 10
...
state = [1, 3, 6], sum = 10, m = 10
```

Sweet, we got our first sum. Then we continue until we reach the state of `[6, 6, 6]`.

We can implement this approach somewhat similarly to the code below[^1]:

```go
func Naive(s, n, m int) int {
  values := make([]int, s)
  for i := 0; i < len(values); i++ {
    values[i] = i + 1
  }

  state := make([]int, n)
  for i := 0; i < len(state); i++ {
    state[i] = 1
  }

  return bruteforce(values, state, n, 0, m)
}

func bruteforce(values, state []int, limit, index, m int) int {
  result := 0
  if index == limit {
    if sum(state) == m {
      result++
    }
  } else {
    for i := 0; i < len(values); i++ {
      state[index] = values[i]
      result += bruteforce(values, state, limit, index+1, m)
    }
  }
  return result
}

func sum(state []int) int {
  result := 0
  for i := 0; i < len(state); i++ {
    result += state[i]
  }
  return result
}
```

The `Naive` method creates and initializes a set of possible die values, `values`, and an initial state of the rolls, `state`. Then `bruteforce` does exactly what we described above: it iterates over all possible combinations of the `state` and checks if their sums equal to `m`.

This is probably not the best brute-force implementation, but we won't go further with it. Even if we start optimizing it somehow, the underlying logic is so slow that it's unlikely we'll gain significant performance.

## Smart Brute-Force

I shared the problem with my friend [Alexei](https://github.com/alexeimatrosov), who is not only a smart guy but also has a grip on efficient algorithms. He came up with a smarter version of the brute-forcing approach:

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

Here, we decrease possibilities of obtaining a sum `m` by iterating over the roll values and narrowing down a possible number of remaining rolls and their sums. We start by enumerating all possible die values, from `1` to `s`. If the first roll gives us `1`, then the next `n - 1` rolls should give us a sum of `m - 1`. Next, if the second roll gave us `1`, the next `n - 2` rolls would have to be `m - 2`. Finally, we either hit the final roll or discard the rest of the rolls because they can't give us the desired sum.

Let's see how it works using the same setup as in the example above:

```
s = 6, n = 3, m = 10
```

We start with the first roll giving us `1`, then the two remaining rolls have to be a sum of `9`:

```
i = 1, n = 2, m = 9
```

We move on to the second roll and start with `1` again:

```
i = 1, n = 1, m = 8
```

We clearly see that it's not possible for the third and final roll to have a value of `8`. Hence, we discard this branch by returning `0`. We repeat the logic recursively until `i` reaches `s`.

Let's do some benchmarking by rolling a 6-sided die 10 times and counting sums of 31:

```
BenchmarkNaive                3   425672694 ns/op
BenchmarkSmart               72    15923781 ns/op
```

Wow, leagues ahead of the naive brute-force.

## Binomial Coefficients

In both implementations, we iterate over all possible roll combinations which is not very efficient. Wouldn't it be nice to have a deterministic formula for calculating probability without enumerating all possible die combinations? Turns out, such a formula exists, but we have to work out the math in order to see where it's coming from.

As you might have realized already, rolling a die and analyzing its results fall into the category of statistics and probability. Luckily, the problem is fairly known in the field, so we can rely on existing analysis. We'll go through an [article on dice](https://mathworld.wolfram.com/Dice.html) by Eric Weisstein that dissects the probability of dice rolls and will highlight key points from there. In the end, we'll get a nice and simple formula that solves the problem.

First, we need to define a [probability-generating function](https://en.wikipedia.org/wiki/Probability-generating_function) for a single die roll:

$$ f(x) = x^1 + x^2 + ... + x^s $$

Where `s` is the number of sides on the die, the exponent represents the score on the die face, and the coefficient indicates the number of ways each score can be obtained. We can also write the series as a closed-form formula:

$$ f(x) = x \dfrac{1-x^s}{1-x} $$

Next, to adjust the formula to `n` rolls we need to exponentiate the whole expression:

$$ f(x) = x^n (\dfrac{1-x^s}{1-x})^n $$

Or, putting it in a more simple form:

$$ f(x) = x^n (1-x^s)^n (1-x)^{-n} $$

Now we have two binomials in the power of `n` and `-n`, respectively, that we need to crack. Luckily, we have the [binomial theorem](https://en.wikipedia.org/wiki/Binomial_theorem) that comes to the rescue. Without diving too much into the math, let's note that the theorem allows us to convert the power of a binomial into a more simple sum:

$$ (1+x)^n = \sum_{\mathclap{0\le k\le n}} \dbinom n k x^k $$

The expression inside the parenthesis is called a [binomial coefficient](https://en.wikipedia.org/wiki/Binomial_coefficient), which is fairly easy to calculate:

$$ \dbinom n k = \dfrac{n!}{k!(n - k)!} $$

Coming back to our probability-generating function, we can use the binomial theorem to unravel the expression:

$$ f(x) = x^n ( \sum_{\mathclap{0\le i\le n}} \dbinom n i (-1)^i x^{si} ) ( \sum_{\mathclap{0\le j\le \infin}} \dbinom {-n} j (-1)^j x^j ) $$

Now, the expression includes all possible die sums. However, we don't need all of them, we only need the ones that are equal to `m`. Coefficient, responsible for the number of ways to obtain a sum equal to `m`, is `x^m`, thus we only need to include the terms for which `m = n + s * i + j`[^2].

So we finally get the exact formula for our problem:

$$ f(x, m) = \displaystyle\sum_{m = n + si + j} (-1)^{i+j} \dbinom n i \dbinom {-n} j $$

Where we sum over non-negative `i` and `j` for which `m = n + s * i + j`.

Let's pull our familiar example of a 6-sided die being rolled 3 times and count the sums of 10 using this formula.

First, we need to find the number of terms using `i` and `j`. We begin with replacing `m`, `n`, and `s` in the equation:

```
10 = 3 + 6 * i + j
```

Since both `i` and `j` are non-negative, we start with `i = 0`:

```
10 = 3 + 6 * 0 + j
10 = 3 + j
j = 7
```

Then we increment `i`, so now `i = 1`:

```
10 = 3 + 6 * 1 + j
10 = 9 + j
j = 1
```

Finally, we try `i = 2`:
```
10 = 3 + 6 * 2 + j
10 = 15 + j
j = -3
```

Which breaks the contract. So we only have two terms, `i = 0, j = 7` and `i = 1, j = 1`:

$$ f(x, 10) = (-1) \dbinom 3 0 \dbinom {-3} 7 + \dbinom 3 1 \dbinom {-3} 1 $$

Before we calculate the expression, let's briefly note that in the second term we have a binomial coefficient with negative `n` which can be [written]((https://en.wikipedia.org/wiki/Binomial_coefficient#Generalization_to_negative_integers_n)) as:

$$ \dbinom {-n} k = (-1)^k \dbinom{n + k - 1}{k} $$

So our two-terms expression can be computed like that:

$$ f(x, 10) = (-1) \dfrac{3!}{0!(3 - 0)!} (-1) \dfrac{9!}{7!(9 - 7)!} + \dfrac{3!}{1!(3 - 1)!} \dfrac{3!}{1!(3 - 1)!} = (-1) (-18) + 9 = 27 $$

By rolling a 6-sides die 3 times, we can obtain a sum of 10 in 27 ways.

## Binomial Implementation

Enough math, now it's finally time to jump into a code editor. We start with the main `Binomial` method:

```go
func Binomial(s, n, m int) int {
  result := 0
  i := 0
  for s*i <= m-n {
    j := m - n - i*s
    result += sign(i+j) * coef(n, i) * sign(j) * coef(n+j-1, j)
    i++
  }
  return result
}

func sign(n int) int {
  if n%2 == 0 {
    return 1
  }
  return -1
}
```

First, we need to find `i` and `j`. Once we find their values, we compute a sign. In the formula, we do that by exponentiating `-1` to the power of `i + j`. However, this calculation would be more expensive for the processor, so we just use a good old `mod` operation. Then, we compute binomial coefficients for two pairs, `n, i` and `-n, j`. To calculate the negative `n` binomial, we do the switch we described above: we replace it with a binomial of `n + j - 1, j` multiplied by `(-1)^j`.

Next, let's have a look at the `coef` method that calculates a binomial coefficient:

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

Normally, a binomial coefficient is computed using factorials. However, they are notorious for being large, so once we get over factorial of 12 we overflow the size of `int`. For that purpose alone, we will switch from computing factorials to iterating over `k` or `n - k` – whichever is the minimum – and building up the coefficient by calculating intermediate results.

Finally, we can check if we improved the algorithm performance-wise:

```
BenchmarkNaive                3   425482639 ns/op
BenchmarkSmart               73    15905076 ns/op
BenchmarkBinomial      25150377       47.25 ns/op
```

And we indeed did.

You can check out the code from this post on GitHub: [dnd](http://github.com/timiskhakov/dnd).

## Further Reading

- [The dice roll with a given sum problem](https://www.lucamoroni.it/the-dice-roll-sum-problem)
- [Binomial Distribution](https://onlinestatbook.com/2/probability/binomial.html)
- [Binomial distributions, Probabilities of probabilities, part 1](https://www.youtube.com/watch?v=8idr1WZ1A7Q)

## Footnotes

[^1]: I've been writing a lot of Go code lately, so I'll stick with Go in this post. Although, it shouldn't be hard to port the code to other languages.

[^2]: To be honest, I'm quite puzzled by this inference. The author of the article doesn't explain where it came from or how he made this conclusion. However, it works!